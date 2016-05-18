# Hexo

### cmd

- hexo init
- hexo new 
- hexo clean
- hexo g
- hexo d
- hexo new page xxx
- hexo new [tm] xxx


- Git 安装hexo-deployer-git

	$ npm install hexo-deployer-git --save

修改配置。

	deploy:
  		type: git
  		repo: <repository url>
		branch: [branch]
  		message: [message]


