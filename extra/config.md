###### 下载blog源文件
git clone https://github.com/flyingeagle1015/flyingeagle1015.github.io kbase

###### ssh key配置
```shell
cd kbase/extra;   
#解压ssh publickey
#tar -zcf  - filename |openssl des3 -salt | dd of=id_rsa_github.des3
dd if=id_rsa_github.des3 |openssl des3 -d | tar zxf -
cp id_rsa_github.des3 $HOME/.ssh/
cat >> $HOME/.ssh/config <<EOF
# github
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/id_rsa_github
User L
EOF 

#测试
ssh -T git@github.com
```

###### 安装hexo相关包
首先需要安装hexo（如果安装失败，切换到root后安装）：  
sudo npm install hexo -g  
sudo npm install -g cnpm --registry=https://registry.npm.taobao.org  
cd kbase; npm install

###### 写新博客
hexo new

###### 生成博客
hexo generate

###### 预览博客
hexo server  
访问本地服务器http://localhost:4000预览  

###### 发布博客
hexo deploy

###### 提交博文
git commit -a -m 'new pages'; git push;

###### 采用SSH key免密码提交
git remote rm origin
git remote add origin git@github.com:github_username/github_project.git
git push origin

