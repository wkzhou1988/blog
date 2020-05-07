---
title: Hello World dsfsdafsdfdsa
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start dsfasfds

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)

```csharp
public class Solution {
    bool compare(TreeNode s, TreeNode t)
    {
        if (s == null && t == null)
            return true;

        if (!(s != null && t != null))
            return false;

        if (s.val == t.val && compare(s.left, t.left) && compare(s.right, t.right))
            return true;
        return false;
    }

    public bool IsSubtree(TreeNode s, TreeNode t)
    {
        if (compare(s, t))
            return true;
        else if (s.left != null && IsSubtree(s.left, t))
            return true;
        else if (s.right != null && IsSubtree(s.right, t))
            return true;
        return false;
    }
}
```
