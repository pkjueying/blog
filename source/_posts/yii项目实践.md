---
title: yii项目实践
toc: true
comment: true
date: 2016-05-18 15:49:18
categories: [old blog]
tags: [Yii]
description: 在mobileportal项目中使用Yii总结

---


### 分页  

```php

<?php  
$criteria = new CDbCriteria();  
$criteria->order = ' ctimedesc';         //按什么字段来排序  
$count =NewsComments::model()->count($criteria);//count()函数计算数组中的单元数目或对象中的属性个数。  
$pager = new CPagination($count);  
$pager -> pageSize =5;                           //每页显示的行数  
$pager->applyLimit($criteria);  
$newsCommentList =NewsComments::model()->findAll($criteria);//查询所有的数据  
  
$this->render('view' , array(  
    'pages'=>$pager,  
    'list'=>$newsCommentList,  
  ));  
?>  
//然后在view视图中：  
<?php  
 $this->widget('CLinkPager',array(  
  'header'=>'',  
  'firstPageLabel'=>'首页',  
  'lastPageLabel'=>'末页',  
  'prevPageLabel'=>'上一页',  
  'nextPageLabel'=>'下一页',  
  'pages'=>$pages,  
  'maxButtonCount'=>13,  
 ));  
?>  
```
  
### 在时间段中查找相应数据  
```php 

<?php  
//在Models的search()函数中添加  
$criteria->compare('mtime','>='.$this->ctime,true);    
$criteria->compare('mtime','<='.$this->mtime,true);  
?>  
//例如与第三方时间控件进行整合时，在_search视图中使用代码如下：  
 
 <?php echo $form->label($model,'ctime');?>  
 <?php  
  $this->widget('application.extensions.timepicker.timepicker',array(  
    'model'=>$model,  
    'name'=>'ctime',  
  ));  
 ?>  
  
 <?php  
  $this->widget('application.extensions.timepicker.timepicker',array(  
    'model'=>$model,  
    'name'=>'mtime',  
  ));  
 ?>  
  
```
  
  
### 对查询的数据进行排序并显示

可以在Models的search()方法中添加如下代码:  

```php

<?php>  
return new CActiveDataProvider(get_class($this), array(  
  'pagination'=>array(  
    'pageSize'=>20,//设置每页显示20条  
  ),  
  'sort'=>array(  
    'defaultOrder'=>'comment_idDESC', //按指定的字段进行排序  
  ),  
  'criteria'=>$criteria,  
));  
?>  
```
  
### 在视图中显示非数据库表中的数据

//a.首先在视图中将对应的字段进行替换。如  
```php

<?php  
//   'status',  
array(  
  'name'=>'status',  
  'type'=>'raw',  
  'value'=>array($this,'showStatus')  
),  
?>
```
  
//b.然后在对应的控制器中写相应的方法。如  
```php

<?php  
public function showStatus($data, $row, $c)  
{  
 switch ($data->status)  
 {  
  case 'ready':  
   return'准备';  
  case 'locked':  
   return '锁定';  
  case 'open':  
   return'打开';  
  case 'removed':  
   return'删除';  
 }  
}  
?>  
```
  
### 修改admin视图下默认的CButtonColumn

```php
 
<?php  
array(  
 'class'=>'CButtonColumn',  
 'template'=>'{comment} {view}{update} {delete}',  
 'buttons'=>array(  
  'comment' =>array(  
   'label'=>'评论',  
   'imageUrl'=>Yii::app()->request->baseUrl.'/images/icons/coins.png',  
   'url'=>'Yii::app()->createUrl("newsComments/index",array("news_id"=>$data->news_id, ))',//始终使用$data来获取相关的数据。  
  ),  
 ),  
 'htmlOptions' => array(  
   'style'=>'width:100px',  
 ),  
),  
?>  
```
  
### ckfinder

需要调用ckfinder直接弹出上传文件的相关目录，以便可以选择特定的图片，并将该图片的相关地址存入文本框中。一般在_from视图中进行数据的创建以及更新时可以用到。

```javascript  

<script type="text/javascript"src="../pkjueying/ckfinder/ckfinder.js"></script>  
<script type="text/javascript">  
function BrowseServer(imgId)  
{  
 var finder = new CKFinder() ;  
 finder.basePath = '../pkjueying/ckfinder/';//导入CKFinder的路径  
 finder.selectActionFunction = SetFileField;//设置文件被选中时的函数  
 finder.selectActionData = imgId; //接收地址的inputID  
 finder.popup() ;  
}  

function SetFileField(fileUrl,data)  
{  
 document.getElementByIdx_x_x(data["selectActionData"]).value= fileUrl ;  
}  
</script>  
```
  
```html

<div class="row">  
 <?php echo$form->labelEx($model,'editor_avatar');?>  
 <?php echo$form->textField($model ,'editor_avatar' ,array('id'=>'editor_avatar', ));?>  
 <input type="button" value=" 浏 览 "onclick="BrowseServer('editor_avatar');" />  
 <?php echo$form->error($model,'editor_avatar');?>  
</div>  
```  

### 使用第三方ckeditor+ckfinder控件

// 示例代码如下：  
```php

<?php  
echo $form->labelEx($model,'summary');  
$form->widget('application.extensions.editor.CKkceditor',array(  
  "model" =>$model, // 数据模型  
  "attribute" =>'summary', // 文本域中的字段，也就是之前文本域的名字  
  "height" =>'200px', // 编辑器的高度  
  "width" =>'80%',         //编辑器的宽度  
  "filespath"=>SITE_PATH."www/data/upload",  
  "filesurl"=>Yii::app()->baseUrl."/data/upload",  
  )  
);  
echo $form->error($model,'summary');  
?>  
```
  
### 由数据库表生成所需的代码
```php

<?php  
'modules' =>   array(  
 'gii'=>array(  
  'class'=>'system.gii.GiiModule',  
  'password'=>'pkjueying',  
  // If removed, Gii defaults tolocalhost only. Edit carefully to taste.  
  'ipFilters'=>array('10.10.16.43','10.10.16.18','10.10.16.47','::1'),  
 ),  
),  
?>  
```
  
### 由Yii生成静态页面  
```php

<?php  
//在action函数中修改函数的参数，添加第三个参数，设置为true.思路如下：  
$out_file = $this->render($view,$data,true);  
save_to_html($path, $out_file);//此函数仅仅是示例，具体实现自己写。把$out_file存到指定目录，自己命名  
unset($outFile);  
?>  
```
  
### yii的controller中外挂action  
```php

<?php  
class UpdateAction extends CAction {   
  public function run() {   
 // place the action logichere   
  }   
}  
  
class PostController extends CController{   
  public function actions(){   
 return array('edit'=>'application.controllers.post.UpdateAction',);   
  }   
  ....  
}   
?>  
```
  
### 如何使用theme  
```php

<?php  
//在main.php 里面配置  
return array(  
  'theme'=>'basic',  
  //......  
);  
//要使用theme里面的资源的话，比如说images, js, css, 应该这样，  
Yii::app()->theme->baseUrl.”/images/FileName.gif”  
Yii::app()->Theme->baseUrl.”/css/default/common.css”  
  
?>

```
  
### 在当前页面注册css和js文件  

```php 

<?php  
 $cs=Yii::app()->clientScript;  
 $cs->registerCssFile($cssFile);  
 $cs->registerScriptFile($jsFile);  
?>  
```
  
### 使用widget方式。 
```php
 
//a.显示详细信息  
<?php  
$this->widget('zii.widgets.CDetailView',array(   
    'data'=> $model,   
    'attributes'=> array(   
       'id',   
       'title',   
       'content',   
   ),   
);  
?>  

//b.显示列表，可以进行条件限制和分页  
<?php  
//controller中  
$dataProvider = new CActiveDataProvider('Post',array(   
    'criteria'=> array(   
           'condition' => 'project_id =:project_id',   
           'params' => array(':project_id' =>$pid),   
       ),   
    'pagination'=> array(   
       'pageSize' => '5',   
   ),   
));  
//视图中  
$this->widget('zii.widgets.CListView',array(   
 'dataProvider' => $dataProvider,//数据源   
 'itemView' => '_view',//渲染子视图，传给模板的值用$data接收。   
 ),   
);   
?>  
  
//c.显示列表，但是结果会在表格中显示  
<?php  
   $this->widget('zii.widgets.grid.CGridView',array(   
       'dataProvider'=>$dataProvider,//数据源   
       'columns'=>array(   
           'title',         // display the 'title' attribute   
           'category.name',  // display the 'name' attributeof the 'category' relation//显示与category相关的name   
           'content:html',   // display the'content' attribute as purified HTML显示净化过的HTML格式   
           array(           // display 'create_time' using anexpression   
               'name'=>'create_time',   
               'value'=>'date("M j, Y",$data->create_time)',   
           ),   
           array(           // display 'author.username' using anexpression   
               'name'=>'authorName',   
               'value'=>'$data->author->username',   
           ),   
   array(   //display the 'status' attribute of controller's functionshowStatus($data, $row, $c)  
    'name'=>'status',  
    'type'=>'raw',  
    'value'=>array($this,'showStatus')  
   ),  
           array(           // display a column with "view", "update" and "delete"buttons   
               'class'=>'CButtonColumn',   
           ),   
       ),   
       'filter'=>$model,//对用户的输入进行过滤   
   ));   
?>  
```
  
  
### urlManager的配置  

//a.apache下：在config/main.php的components节点下增加：  
```php

<?php  
 'urlManager'=>array(  
  'urlFormat'=>'path',        
  'rules'=>array(  
   '<controller:\w+>/<id:\d+>'=>'<controller>/view',  
   '<controller:\w+>/<action:\w+>/<id:\d+>'=>'<controller>/<action>',  
   '<controller:\w+>/<action:\w+>'=>'<controller>/<action>',  
  ),  
 ),  
?>  
```

//b.apache配置：  

在app的根目录(项目目录，同入口index.php)创建.htaccess文件。内容如下：  

Options +FollowSymLinks  
IndexIgnore */*  
RewriteEngine on  
\# if a directory or a file exists, use it directly  
RewriteCond %{REQUEST_FILENAME} !-f  
RewriteCond %{REQUEST_FILENAME} !-d  
\# otherwise forward it to index.php  
RewriteRule . index.php  
  
//c.nginx下的配置  
//在config/main.php的components节点下增加：  
```php

<?php  
 'urlManager'=>array(  
  'urlFormat'=>'path',        
  'rules'=>array(  
   '<controller:\w+>/<id:\d+>'=>'<controller>/view',  
   '<controller:\w+>/<action:\w+>/<id:\d+>'=>'<controller>/<action>',  
   '<controller:\w+>/<action:\w+>'=>'<controller>/<action>',  
  ),  
 ),  
?>  
```
//step2：  
//在nginx.conf的server 段添加:  
location / {   
    if (!-e$request_filename){   
       rewrite ^/(.*) /index.php last;   
   }   
}  
