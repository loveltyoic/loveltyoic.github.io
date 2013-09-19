---
layout: post
title: "在Rails4.0中使用Jquery-File-Upload上传多个文件"
date: 2013-09-19 09:45
comments: true
categories: rails mongoid
---
在网站中上传图片是一个非常普遍的需求。感谢强大的[Jquery-File-Upload]，让这个功能简化了许多。
需要说明，我的项目环境是`Rails4.0+Mongoid+carrierwave`，需求是对于一辆汽车，上传多张图片并展示出来。
下面就记录一下我是如何实现的。

<!-- more -->

###Model

```ruby picture.rb
class Picture
  include Mongoid::Document

  belongs_to :car
  mount_uploader :image, PictureUploader # 挂载carrierwave

  field :image, type: String
  field :image_cache, type: String
  field :car_token, type: String
  
  def output_json
    {
      "name" => read_attribute(:image),
      "size" => image.size,
      "url" => image.url,
      "delete_url" => id,
      "picture_id" => id,
      "delete_type" => "DELETE"
    }.to_json
  end

end
```

```ruby car.rb
class Car
  include Mongoid::Document

  has_many :pictures, autosave: true

  def generate_token
    self.token = loop do
      random_token = SecureRandom.urlsafe_base64
      break random_token if Car.find(token: random_token).nil?
    end
  end
end
```
说明：一辆车可以有多张图片，因此用has_many关联。
那么这个`generate_token`是做什么的呢，一会在controller中就会看到用处了！
###Controller
* 因为Rails4.0应用`strong parameters`, 因此需要在控制器中做白名单处理，不然参数会被禁止传入。
```ruby
class PicturesController < ApplicationController

  def destroy
    car = Car.find(params[:car_id])
    @pic = car.pictures.find(params[:id])
    @pic.destroy
    respond_to do |format|
      format.js
    end
  end

  def create
    @picture = Picture.new(pic_params)
    if @picture.save
      respond_to do |format|
        format.html {
          render :json => @picture.output_json,
          :content_type => 'text/html',
          :layout => false
        }
        format.json {
          render :json => @picture.output_json
        }
      end
    else
      render :json => [{:error => "custom_failure"}], :status => 304
    end
  end

  def pic_params
    params.require(:picture).permit(:image, :image_cache, :car_token)
  end

end
```
* 对于CarsController,只节选关键的`new`和`create`。
```ruby
  def new
    @car = Car.new
    @car.generate_token
  end

  def create
    @car = Car.new(car_params)
    @car.pictures << Picture.where(car_token: @car.token)
    @car.user_id = current_user.id
    respond_to do |format|
      if @car.save!
        flash[:success] = '车辆信息创建成功!'
        format.html { redirect_to @car }
        format.json { render json: @car, status: :created, location: @car }
      else
        format.html { render action: "new" }
        format.json { render json: @car.errors, status: :unprocessable_entity }
      end
    end
  end
  def car_params
    params.require(:car).permit(:token)
  end
```
###View表单
```erb
<%= simple_form_for @car, :html => {:multipart => true} do |f| %>
    <%= f.input :token, as: :hidden, value: @car.token %>
    <%= f.submit "保存", class: 'btn btn-primary btn-large' %>
<% end %>
<%= form_for Picture.new, :html => {:multipart => true, id: 'new_picture'} do |f| %>
  <%= f.hidden_field :car_token, value: @car.token %>
  <%= f.file_field :image, multiple: true, name: 'picture[image]' %>
<% end %>
<script>
$(function () {
    $('#new_picture').fileupload({ #调用Jquery-File-Upload
      dataType: 'json',
      progressall: function (e, data) {
        var progress = parseInt(data.loaded / data.total * 100, 10);
        console.log(progress);
        $('#progress .progress-bar').css(
            'width', progress + '%'
        );
      },
      done: function (e, data) {
        
      }
    });
});
</script>
```
###说明
从表单代码可以看出，这里是在一个页面中放了两个表单，一个是car的，另一个是picture的。

图片用[Jquery-File-Upload]上传，实际上是调用了jquery的ajax。

当一次上传多张图片时，实际上是**用ajax将图片一张接一张的上传**。

举例来说，如果我一次上传了3张图片，那么就有3个ajax请求，每一次请求都会触发PicturesController的`create action`。

此时的数据库中，就有了3个picture对象，也就是3张图片。

那么，怎样才能将这3张图片与表单中的`@car`关联起来呢？

一般来说，当我们上传图片时，父对象car还没有save到数据库中。

因此就需要一个域将car与picture关联起来，其实就是额外构造的外键 —— car中的token。

在car的`new action`中，通过generate_token构造外键。

然后在异步上传图片后，在picture的`create action`中存储这个token。

当用户填写表单其他部分并提交后，触发car的`create action`，此时根据token在数据库中查找对应的picture，加入到`car.pictures`队列，至此picture就和car关联起来了。

###写在最后
被上传的问题困扰了一阵子，在看了许多教程和代码后，终于是初步完成了，感谢github上开源代码的前辈！

抱歉我的这个项目并不开源，不过跟本文相关的代码也都贴出来了。
另外我还想弄一个乐高爱好者的网站，那个项目会是开源的。

以上重点说明的都是我觉得开始没弄明白的问题，主要是model怎么设计的，controller怎么执行的，view怎么构造的。

至于[Jquery-File-Upload]怎么用，我觉得主要还是看项目主页上的说明吧，我暂时也就是用了basic的功能。

如果这篇文章能给任何人带来帮助，那么我会非常开心。
[Jquery-File-Upload]: https://github.com/blueimp/jQuery-File-Upload
