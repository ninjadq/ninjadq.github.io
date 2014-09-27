---
layout: post
title: 用rspec来写controller的测试
category: tech
tags: [ruby, rails, rspec, test, controller]
---
## Controller 测试基础

这是一个测试的例子：

```ruby
it "redirects to the home page upon save" do
  post :create, contact: Factory.attributes_for(:contact) 
  response.should redirect_to root_url 
end
```

* 例子的描述是用简明的语言描写
* 这个例子之做一件事情：当发送post请求后，将会返回一个redirect给浏览器
* 一个facotry生成一个传递到controller 方法的测试数据；这里使用的是FactoryGirl的`attribute_for`选项，它会生成一个hash而不是一个ruby object。
* controller spec的最基本的语法就是一个他的REST method（这里是post），和控制器方法（：create），传递给controller spec的参数是可选项。

## 测试用例的组织就结构
首先使用spec写出测试的轮廓，写出一些你需要测试的东西，下面是代码

```ruby
# spec/controllers/contacts_controller_spec.rb 
require 'spec_helper' 
describe ContactsController do 
  describe "GET #index" do 
    it "populates an array of contacts" 
    it "renders the :index view" 
  end 
  describe "GET #show" do 
    it "assigns the requested contact to @contact" 
    it "renders the :show template" 
  end 
  describe "GET #new" do 
    it "assigns a new Contact to @contact" 
    it "renders the :new template" 
  end 
  describe "POST #create" do 
    context "with valid attributes" do 
      it "saves the new contact in the database" 
      it "redirects to the home page" 
    end 
    context "with invalid attributes" do 
      it "does not save the new contact in the database" 
      it "re-renders the :new template" 
    end 
  end 
end
```

* 其中使用`describe`和`context`块来组织清晰的测试用例的结构。
* 然后测试happy path（用户传递给了控制器合法的属性）和 sad path（用户传递给controller的属性不合法或者不完整）。
* 如果应用包含搜权的层，需要额外增加一个contex，即需要测试with和without logged-in user，或者基于用户在系统中不同的角色来组织

## 设置数据
使用factorygirl可以很好的生成测试数据，首先可以使用factorygirl生成一个合法的数据

```ruby
# spec/factories/contacts.rb
factory :contact do |f| 
  f.firstname { Faker::Name.first_name } 
  f.lastname { Faker::Name.last_name }
end
```
然后下面的factory可以用来生成非法的数据
```ruby
factory :invalid_contact, parent: :contact do |f| 
  f.firstname nil 
end
```
* 可以看到invalid_contact的parant是contact，然后修改了其中的一个属性。


## 测试GET

表中的rails controller中包含四种get方法`#index`，`#show`,`#new`,`#edit`,首先来看一下`get`.

```ruby
# spec/controllers/contacts_controller_spec.rb 
# ... other describe blocks omitted omitted 
describe "GET #index" do 
  it "populates an array of contacts" do 
    contact = Factory(:contact) 
    get :index 
    assigns(:contacts).should eq([contact]) 
  end 
  
  it "renders the :index view" do 
    get :index 
    response.should render_template :index 
    end 
  end  
  
describe "GET #show" do 
  it "assigns the requested contact to @contact" do 
    contact = Factory(:contact) 
    get :show, id: contact 
    assigns(:contact).should eq(contact)   
  end 
  it "renders the #show view" do 
    get :show, id: Factory(:contact) 
    response.should render_template :show 
  end 
end 
```

`new`部分略微复杂一点,它building了一新的电话号码将嵌套在contact信息中.

```ruby
# app/controllers/contacts_controller.rb 
# ... other code omitted 
def new 
  @contact = Contact.new %w(home office mobile).each do |phone| 
    @contact.phones.build(phone_type: phone) 
  end 
end 
```

测试如下

```ruby
# spec/controllers/contacts_controller_spec.rb 
# ... other specs omitted 
describe "GET #new" do 
  it "assigns a home, office, and mobile phone to the new contact" do 
    get :new 
    assigns(:contact).phones.map{ |p| p.phone_type }.should eq %w(home office mobile)    
  end 
end
```

## 测试Post
接下来测试控制器的`:create`方法，代码如下：

```ruby
# spec/controllers/contacts_controller_spec.rb 
# rest of spec omitted ... 
describe "POST create" do 
  context "with valid attributes" do 
    it "creates a new contact" do 
      expect{ 
      post :create, contact: Factory.attributes_for(:contact) 
      }.to change(Contact,:count).by(1) 
    end 
    it "redirects to the new contact" do 
      post :create, contact: Factory.attributes_for(:contact) 
      response.should redirect_to Contact.last 
    end 
  end 
  context "with invalid attributes" do 
    it "does not save the new contact" do 
      expect{ 
        post :create, contact: Factory.attributes_for(:invalid_contact) 
      }.to_not change(Contact,:count) 
    end 
    it "re-renders the new method" do 
      post :create, contact: Factory.attributes_for(:invalid_contact) 
      response.should render_template :new 
    end 
  end 
end
```

## 测试PUT
测试控制器的`update`方法，首先需要检查一系列GET方法传进来想要更新的参数，然后重定向到我们所需要的。接着需要测试一下，在`ivalid_attribute`方法下是不会发生的。

```ruby
# spec/controllers/contacts_controller_spec.rb 
# rest of spec omitted ... 
describe 'PUT update' 
do before :each do 
  @contact = Factory(:contact, firstname: "Lawrence", lastname: "Smith") 
end 
  context "valid attributes" do 
    it "located the requested @contact" do 
      put :update, id: @contact, contact: Factory.attributes_for(:contact) 
      assigns(:contact).should eq(@contact) 
    end 
    
    it "changes @contact's attributes" do 
      put :update, id: @contact, 
        contact: Factory.attributes_for(:contact, firstname: "Larry", lastname: "Smith") 
      @contact.reload 
      @contact.firstname.should eq("Larry") 
      @contact.lastname.should eq("Smith") 
    end 
  
    it "redirects to the updated contact" do 
      put :update, id: @contact, contact: Factory.attributes_for(:contact) 
      response.should redirect_to @contact 
    end 
  end 
  context "invalid attributes" do 
    it "locates the requested @contact" do 
      put :update, id: @contact, contact: Factory.attributes_for(:invalid_contact)
      assigns(:contact).should eq(@contact) 
    end 
    it "does not change @contact's attributes" do 
      put :update, id: @contact, contact: Factory.attributes_for(:contact, firstname: "Larry", lastname: nil) 
      @contact.reload 
      @contact.firstname.should_not eq("Larry") 
      @contact.lastname.should eq("Smith") 
    end 
    it "re-renders the edit method" do 
      put :update, id: @contact, contact: Factory.attributes_for(:invalid_contact)
      response.should render_template :edit 
    end 
  end 
end 
# rest of spec omitted 
```
* 注意对`@contact`使用`reload`方法，来保证我们的update是持久化的

## 测试DELETE方法
`destroy`方法对应delete

```ruby
# spec/controllers/contacts_controller_spec.rb 
# rest of spec omitted ... 
describe 'DELETE destroy' do 
  before :each do 
  @contact = Factory(:contact) 
  end 
  it "deletes the contact" do 
    expect{ 
      delete :destroy, id: @contact 
    }.to change(Contact,:count).by(-1) 
  end 
  it "redirects to contacts#index" do 
    delete :destroy, id: @contact 
    response.should redirect_to contacts_url 
  end 
end 
# rest of spec omitted 
```