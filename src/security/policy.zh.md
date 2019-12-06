# Policy

安全是我们所做工作的核心原则之一，为此，我们要确保 async-std 具有安全的实现。感谢您抽出宝贵的时间，透露您发现的任何问题。

async-std 发行版中，所有安全错误应通过电子邮件报告至 florian.gilcher@ferrous-systems.com 。列表已交付给小型安全团队。我们会在 24 小时内确认您的电子邮件，并且会在 48 小时内，收到对电子邮件的详细回复，指示您处理报告的后续步骤。如果您愿意，您可以使用我们的公钥加密报告。该密钥也位于 MIT 的密钥服务器上，复制如下。

确保使用一行，来描述主题，以免你的报告被错过。在对您的报告进行初次答复后，安全团队努力使您，了解有关修复和完整公告的进展情况。正如[RFPolicy][rf-policy]所推荐的，这些更新至少每五天发送一次。实际上，更有可能是每 24-48 小时一次。

如果您在 48 小时内，未收到对电子邮件的答复，或者在过去五天内没有收到安全团队的来信，则可以采取一些步骤（按顺序）：

- 在我们的社区论坛上发帖

请注意，讨论论坛是公共区域。在错误补丁更新过程，请勿讨论您的 issue。只需说，您正在设法与安全团队联系。

[rf-policy]: https://en.wikipedia.org/wiki/RFPolicy

## Disclosure policy

async-std 项目具有 5 个步骤的公开过程。

- The security report is received and is assigned a primary handler. This person will coordinate the fix and release process.
- The problem is confirmed and a list of all affected versions is determined.
- Code is audited to find any potential similar problems.
- Fixes are prepared for all releases which are still under maintenance. These fixes are not committed to the public repository but rather held locally pending the announcement.
- On the embargo date, the changes are pushed to the public repository and new builds are deployed to crates.io. Within 6 hours, a copy of the advisory will be published on the the async.rs blog.

此过程可能需要一些时间，特别是在需要与其他项目的维护者进行协调时。我们将尽一切努力尽快解决该错误，但是，请务必遵循上面的发布流程，以确保以一致的方式处理该披露，这一点很重要。

## Credits

该政策是根据[Rust project](https://www.rust-lang.org/policies/security)安全政策。

## PGP Key

```text
-----BEGIN PGP PUBLIC KEY BLOCK-----

mQENBF1Wu/ABCADJaGt4HwSlqKB9BGHWYKZj/6mTMbmc29vsEOcCSQKo6myCf9zc
sasWAttep4FAUDX+MJhVbBTSq9M1YVxp33Qh5AF0t9SnJZnbI+BZuGawcHDL01xE
bE+8bcA2+szeTTUZCeWwsaoTd/2qmQKvpUCBQp7uBs/ITO/I2q7+xCGXaOHZwUKc
H8SUBLd35nYFtjXAeejoZVkqG2qEjrc9bkZAwxFXi7Fw94QdkNLaCjNfKxZON/qP
A3WOpyWPr3ERk5C5prjEAvrW8kdqpTRjdmzQjsr8UEXb5GGEOo93N4OLZVQ2mXt9
dfn++GOnOk7sTxvfiDH8Ru5o4zCtKgO+r5/LABEBAAG0UkZsb3JpYW4gR2lsY2hl
ciAoU2VjdXJpdHkgY29udGFjdCBhc3luYy1zdGQpIDxmbG9yaWFuLmdpbGNoZXJA
ZmVycm91cy1zeXN0ZW1zLmNvbT6JATgEEwECACIFAl1Wu/ACGwMGCwkIBwMCBhUI
AgkKCwQWAgMBAh4BAheAAAoJEACXY97PwLtSc0AH/18yvrElVOkG0ADWX7l+JKHH
nMQtYj0Auop8d6TuKBbpwtYhwELrQoITDMV7f2XEnchNsvYxAyBZhIISmXeJboE1
KzZD1O+4QPXRcXhj+QNsKQ680mrgZXgAI2Y4ptIW9Vyw3jiHu/ZVopvDAt4li+up
3fRJGPAvGu+tclpJmA+Xam23cDj89M7/wHHgKIyT59WgFwyCgibL+NHKwg2Unzou
9uyZQnq6hf62sQTWEZIAr9BQpKmluplNIJHDeECWzZoE9ucE2ZXsq5pq9qojsAMK
yRdaFdpBcD/AxtrTKFeXGS7X7LqaljY/IFBEdJOqVNWpqSLjGWqjSLIEsc1AB0K5
AQ0EXVa78AEIAJMxBOEEW+2c3CcjFuUfcRsoBsFH3Vk+GwCbjIpNHq/eAvS1yy2L
u10U5CcT5Xb6be3AeCYv00ZHVbEi6VwoauVCSX8qDjhVzQxvNLgQ1SduobjyF6t8
3M/wTija6NvMKszyw1l2oHepxSMLej1m49DyCDFNiZm5rjQcYnFT4J71syxViqHF
v2fWCheTrHP3wfBAt5zyDet7IZd/EhYAK6xXEwr9nBPjfbaVexm2B8K6hOPNj0Bp
OKm4rcOj7JYlcxrwhMvNnwEue7MqH1oXAsoaC1BW+qs4acp/hHpesweL6Rcg1pED
OJUQd3UvRsqRK0EsorDu0oj5wt6Qp3ZEbPMAEQEAAYkBHwQYAQIACQUCXVa78AIb
DAAKCRAAl2Pez8C7Uv8bB/9scRm2wvzHLbFtcEHaHvlKO1yYfSVqKqJzIKHc7pM2
+szM8JVRTxAbzK5Xih9SB5xlekixxO2UCJI5DkJ/ir/RCcg+/CAQ8iLm2UcYAgJD
TocKiR5gjNAvUDI4tMrDLLdF+7+RCQGc7HBSxFiNBJVGAztGVh1+cQ0zaCX6Tt33
1EQtyRcPID0m6+ip5tCJN0dILC0YcwzXGrSgjB03JqItIyJEucdQz6UB84TIAGku
JJl4tktgD9T7Rb5uzRhHCSbLy89DQVvCcKD4B94ffuDW3HO8n8utDusOiZuG4BUf
WdFy6/gTLNiFbTzkq1BBJQMN1nBwGs1sn63RRgjumZ1N
=dIcF
-----END PGP PUBLIC KEY BLOCK-----
```
