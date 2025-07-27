---
title: 素の OpenJDK 8 だと maven のリポジトリからダウンロードに失敗するので対処した件
tags:
  - Java
  - Maven
  - OpenJDK
private: false
updated_at: '2021-04-16T14:45:08+09:00'
id: de7cfd4b9fe64cbe4045
organization_url_name: null
slide: false
ignorePublish: false
---
:sweat_smile: 今更、OpenJDK 8 ですかというツッコミはなしで…

Oracle JDK から OpenJDK に切り替えてから、maven のセントラルリポジトリから、以下のような警告が出てダウンロードができなくなりました。

---

[WARNING] Failed to retrieve plugin descriptor for org.apache.maven.plugins:maven-clean-plugin:2.5: Plugin org.apache.maven.plugins:maven-clean-plugin:2.5 or one of its dependencies could not be resolved: Failed to read artifact descriptor for org.apache.maven.plugins:maven-clean-plugin:jar:2.5
Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-install-plugin/2.4/maven-install-plugin-2.4.pom

---

で、SSL の問題かと思い、

[maven - MavenにSSLエラーを無視するように指示する方法(およびすべての証明書を信頼する方法) - ITツールウェブ](https://cloud6.net/so/maven/39411)

というページを見つけ、以下のオプションを追加したら、ダウンロードできることは確認できました。

```
-Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true
```

ですが、セキュリティを無視するわけですから、気持ちよくありません。

おそらく https://repo.maven.apache.org/ の証明書を発行している認証局のルート証明書(DigiCert Global Root CA)が OpenJDK のキーストアに入ってないのだろうということで

[Java環境に手動でグローバルサインのルート証明書をインストールする方法 | サポート・お申し込みガイド | GMOグローバルサイン【公式】](https://jp.globalsign.com/support/faq/331.html)

を参考に keytool を実行したのですが、キーストアの初期パスワードが分からないという罠に。

[java - OpenJDK keytool password - Stack Overflow](https://stackoverflow.com/questions/56847306/openjdk-keytool-password)

を見つけ、`changeit` であることが分かりました。

```
keytool -import -alias DigiCertGlobalRootCA -keystore Path\to\java-se-8u41-ri\jre\lib\security\cacerts -file filename.pem
```

無事にキーストアに追加でき、特にオプションを指定しなくても Maven セントラルリポジトリから、ダウンロードできるようになりました。

