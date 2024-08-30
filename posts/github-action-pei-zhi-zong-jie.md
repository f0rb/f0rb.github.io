---
title: 'Github Action 配置总结'
date: 2021-06-13 08:00:02
tags: []
published: false
hideInList: false
feature: 
isTop: false
---

GPG配置

gpg --list-secret-keys --keyid-format=long

需要导出私钥
gpg --export-secret-keys --armor {your_keyId}

maven-gpg-plugin:1.1:sign failed.: NullPointerException}
添加-Dgpg.passphrase=${{ secrets.GPG_PASSPHRASE }}


- gpg: signing failed: Inappropriate ioctl for device
  ```
        env:
          GPG_TTY: $(tty)
  ```
- gpg: signing failed: No such file or directory
  把这段配置加到maven-gpg-plugin的execution下
  ```
  <configuration>
  <!-- Prevent `gpg` from using pinentry programs -->
  <gpgArguments>
    <arg>--pinentry-mode</arg>
    <arg>loopback</arg>
  </gpgArguments>
</configuration>
```
   
- gpg: signing failed: No pinentry
```
gpgconf --kill gpg-agent
echo "test" | gpg --clearsign
```

- 401
 server-id: sonatype-nexus-snapshots # Value of the distributionManagement/repository/id field of the pom.xml
        

https://docs.github.com/cn/actions/guides/publishing-java-packages-with-maven#publishing-packages-to-the-maven-central-repository

https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification/generating-a-new-gpg-key

添加
https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification/adding-a-new-gpg-key-to-your-github-account


-P sonatype-oss-release

Permission denied (publickey) organization

scm配置从git协议换为https协议
