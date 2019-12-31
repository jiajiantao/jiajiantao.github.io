# <center>gitlab中使用``hook``对提交消息进行校验</center>

## 全局设置（对每个仓库都生效）

- 在``/etc/gitlab/gitlab.rb``中修改配置

```
gitlab_shell['custom_hooks_dir'] = "/var/opt/gitlab/custom_hooks"
```

- 创建``custom_hooks``目录，并在目录下创建目录``pre-receive.d``
```
mkdir -p /var/opt/gitlab/custom_hooks/pre-receive.d
```

- 在``pre-receive.d``中新建文件pre-receive并赋予可执行权限

- pre-receive的内容如下

```
#!/opt/gitlab/embedded/bin/ruby
#encoding:utf-8
stdin = STDIN.read.split(" ")
last_commit_id = stdin.at(1)
last_push_commit_id = stdin.at(0)
puts "上次提交的commitId为:#{last_commit_id}"
puts "上次推送的commitId为:#{last_push_commit_id}"

header_regex = /^(feat|fix|docs|refactor|test|chore|style|init|merge|revert|version|license|dependency|conf|database|drop|deprecated|performance|context|wip)(\([\w\-]+\))?([:|\：]){1}([\u4e00-\u9fa5_a-zA-Z0-9\s\·\~\！\@\#\￥\%\……\&\*\（\）\——\-\+\=\【\】\{\}\、\|\；\‘\’\：\“\”\《\》\？\，\。\、\`\~\!\#\$\%\^\&\*\(\)\_\[\]{\}\\\|\;\'\'\:\"\"\,\.\/\<\>\?]){1,150}$/
body_regex = /([\u4e00-\u9fa5_a-zA-Z0-9\s\·\~\！\@\#\￥\%\……\&\*\（\）\——\-\+\=\【\】\{\}\、\|\；\‘\’\：\“\”\《\》\？\，\。\、\`\~\!\#\$\%\^\&\*\(\)\_\[\]{\}\\\|\;\'\'\:\"\"\,\.\/\<\>\?]){1,200}$/
footer_regex = /^((BREAKING CHANGE)|(Closes)){1}[\s\S]*$/
z40 = "0000000000000000000000000000000000000000"
missed_revs = `git rev-list #{last_commit_id}`.split("\n")
if z40 != last_push_commit_id
   missed_revs = `git rev-list #{last_push_commit_id}..#{last_commit_id}`.split("\n")
end 
puts "未推送提交:#{missed_revs}"
missed_revs.each do |rev|
  message = `git cat-file commit #{rev} | sed '1,/^$/d'`
  message_splits = message.split(/\n\n/)
  element_size = message_splits.length
  if (element_size < 2)
    puts "commitId为:#{rev}的消息格式错误(需包含header和body并以空行隔开):#{message}"
    exit 1
  elsif (element_size == 3 )
    footer = message_splits.at(2)
    if ( !footer_regex.match(footer.force_encoding("UTF-8")) )
       puts "commitId为:#{rev}的提交消息footer格式错误:#{footer}"
       exit 1
    end
  end
   header = message_splits.at(0)
   body = message_splits.at(1)
  if ( !body_regex.match(body.force_encoding("UTF-8")) )
    puts "commitId为:#{rev}的提交消息body格式错误:#{body}"
    exit 1
  end
  if (!header_regex.match(header.force_encoding("UTF-8")))
    puts "commitId为:#{rev}的提交消息header格式错误:#{header}"
    exit 1
  end
end
```
- git commit message 格式为``https://drive.google.com/file/d/1quTrsgOb5n1WBf7XgFVszRsubbSEiqAs/view?usp=sharing``
## 单个仓库配置
- 在``/var/opt/gitlab/git-data/repositories/<group>/<project>.git``目录下创建文件夹``custom_hooks``和``pre-receive.d``，如

```
mkdir -p /var/opt/gitlab/git-data/repositories/@hashed/4a/44/4a44dc15364204a80fe80e9039455cc1608281820fe2b24f1e5233ade6af1dd5.git/custom_hooks/pre-receive.d
```

- 在``pre-receive.d``中新建文件pre-receive并赋予可执行权限
- pre-receive的内容同上
- [参考链接](https://docs.gitlab.com/ee/administration/custom_hooks.html)
