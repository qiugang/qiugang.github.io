
🏷️设计是近年来很常用的设计，用于展示分类，属性，搜索关键字等等，Github 上也有不少 tag 相关的自定义view, 最常用的还是流式 tag 布局. 基于此能够实现一个像使用 EditText 编辑文字一样编辑 tag 的 view.

 只需保证最后的 child view 是输入 tag 内容的 EditText。因为 tag 显示宽度的问题，不能像文字一样排列显示，需要在横向宽度不够的情况下换行显示。正因为如此，很自然就想到了基于 FlowLayout 来实现，至于具体的 tag 添加删除，只需要针对输入 tag 的 EditText 做相关逻辑处理就可以了.

关于已经存在的 tag 的删除：一直没想到满意的交互方式，微信上删除用户 tag 的是长按出现一个 x 的 icon，表示该 tag 进入了删除状态，然后再点击 x 删除。 为了保持删除操作与 EditText 中删除字符交互差不多的体验，最后采取 pocket 的删除方式（囧，最后实现出来，整个效果是一毛一样的):点击某个 tag 表示选中（相当于移动 EditText 的光标），再点击键盘的 Del 就把选中的 tag 删除.

code 在这里：[https://github.com/qiugang/EditTag](https://github.com/qiugang/EditTag)，欢迎各种 PR, Issues.

