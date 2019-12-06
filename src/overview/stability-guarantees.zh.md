# Stability and SemVer

`async-std`跟随<https://semver.org/>的脚步。

简而言之：我们将软件版本控制为`MAJOR.MINOR.PATCH`。我们补充下：

- MAJOR 版本，API 不兼容
- MINOR 版本，向后兼容地给出功能
- PATCH 版本，向后兼容的错误修复时的

而文档方面，我们将提供主要版本之间的迁移文档。

## Future expectations

`async-std`使用自己的实现，有一下几个概念：

- `Read`
- `Write`
- `Seek`
- `BufRead`
- `Stream`

为了与生态系统集成，实现这些 trait 的所有类型，对应的，也在`futures-rs`箱子有实现。请注意，我们的 SemVer 守则，不会扩展到这些接口的使用。我们希望这些内容是保守更新，并同步进行。

## Minimum version policy

当前的暂行策略是，可以在 minor 版本更新中，要增加使用此箱子所需的最低 Rust 版本。例如，如果`async-std` 1.0 需要 Rust 1.37.0，然后对`async-std` 1.0.z 来说(其中 z 是数字)，就会需要 Rust 1.37.0 或更高版本。然而，对`async-std` 1.y 来说(表示 y> 0)，那么你可能需要更新的 Rust 版本。

通常，此箱子比较保守，会遵循最低支持版本的 Rust。不过，`async/await`毕竟是作为一项新功能，我们会权衡保守与勇进的。

## Security fixes

安全修复程序会给*所有*的 minor 分支上应用，终落在该库的所有*支持的* major 版本上。此策略将来可能会更改，在这种情况下，我们至少会发出通知*3 个月*先。

## Credits

此策略基于[BurntSushi's regex crate][regex-policy]。

[regex-policy]: https://github.com/rust-lang/regex#minimum-rust-version-policy
