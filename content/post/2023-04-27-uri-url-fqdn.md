---
title: 一文搞懂 URI & URL 的差別
subtitle: 
date: 2023-04-27 12:00:00
tags: ["網路"]
---


{{< line_break >}}

URI, URL 這兩個詞是我相當常使用到的詞彙，但具體這兩個差別在哪，我一直沒認真記住過，使用起來都把他們當作"網頁連結"來看，這次藉著筆記就來好好認真記住這兩個的差別！


URI (Uniform Resource Identifier)，統一資源標識符

是用於標誌某一資源的字串，其實不一定限於網路上的資源

<!--more-->

通用的 URI 格式：

[scheme]://[authority]/[path]?[query]#[fragment]

拿這 URI 來說：

foo://example.com:8042/over/there?name=ferret#nose

- scheme = foo
- authority = //example.com:8042
- path = /over/there
- query = name=ferret
- fragment = nose

當中 scheme 和 path 是必須的

以下這些都是 URI：
- `ftp://ftp.is.co.za/rfc/rfc1808.txt`
- `http://www.ietf.org/rfc/rfc2396.txt`
- `ldap://[2001:db8::7]/c=GB?objectClass?one`
- `mailto:John.Doe@example.com`
- `news:comp.infosystems.www.servers.unix`
- `tel:+1-816-555-1212`
- `telnet://192.0.2.16:80/`
- `urn:oasis:names:specification:docbook:dtd:xml:4.1.2`


{{< line_break >}}

URL (Uniform Resource Locator)，統一資源定位符

其實 URL 是 URI 的子集 (即所有 URL 都是一種 URI，URI 則不一定是 URL)

URL 除了標誌某個資源，更提供了這資源的定位方法，所以 URL 側重的是你可以拿這字串來取得資源這件事

像是網頁、FTP、email (mailto)、JDBC 這些都能讓我利用此字串取得該資源，就是可稱為 URL


{{< line_break >}}

