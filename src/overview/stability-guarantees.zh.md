# Stability and SemVer

`async-std`跟随<https://semver.org/>。

简而言之：我们将软件版本控制为`MAJOR.MINOR.PATCH`。我们增加：

-   API版本不兼容时的主要版本，
-   当我们以向后兼容的方式介绍功能时的MINOR版本
-   进行向后兼容的错误修复时的PATCH版本

我们将提供主要版本之间的迁移文档。

## Future expectations

`async-std`使用以下概念的自己的实现：

-   `Read`
-   `Write`
-   `Seek`
-   `BufRead`
-   `Stream`

为了与生态系统集成，实现这些特征的所有类型还必须在`futures-rs`图书馆。请注意，我们的SemVer保证不会扩展到这些接口的使用。我们希望这些内容会保守地更新并同步进行。

## Minimum version policy

当前的暂行策略是，可以在次要版本更新中增加使用此板条箱所需的最低Rust版本。例如，如果`async-std`1.0需要Rust 1.37.0，然后`async-std`所有z值的1.0.z也将需要Rust 1.37.0或更高版本。然而，`async-std`1.y表示y> 0可能需要更新的最低Rust版本。

通常，相对于最低支持版本的Rust，此板条箱比较保守。用`async/await`不过，作为一项新功能，我们将在最初以可衡量的速度跟踪变化。

## Security fixes

安全修复程序将应用于*所有*该库的所有小分支*支持的*主要修订。此政策将来可能会更改，在这种情况下，我们至少会发出通知*3个月*先。

## Credits

此政策基于[BurntSushi's regex crate][regex-policy]。

[regex-policy]: https://github.com/rust-lang/regex#minimum-rust-version-policy
