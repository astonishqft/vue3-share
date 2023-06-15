## 如何使用

简化了，就三个命令：

- new：使用线上模板创建一个新的 md 文件
- serve：启动一个 md 文件的 webpack dev server
- build：编译产出一个 md 文件

```bash
# create a new slide with an official template 
$ nodeppt new slide.md
 
# create a new slide straight from a github template 
$ nodeppt new slide.md -t username/repo
 
# start local sever show slide 
$ nodeppt serve slide.md
 
# to build a slide 
$ nodeppt build slide.md
```