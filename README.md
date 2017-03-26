Validation Helper for form in Mobx
==================================

## Installation

```bash
npm install https://github.com/PizzaLiu/mobx-form-validate.git\#npm --save
```

## API

### @validate(regexp[, message])
### @validate((value, this)=>string|undefined)

Use this decorator on your properties, which must be aslo decorated by @observable or @computed.

This will add a hidden `@computed` field named with 'validateError' prefix in camel-case, 
which should be undefined when validation successed, or be a string which indicate error message.

This will also add a computed field named 'validateError', which indicate any error occuered in this form.

This will also add a computed field named 'isValid', whether this form has no validation error.

### @validate(regexpArr, messageArr)


## Usage

### Create a observable class with validation

```js
import {observable, computed} from 'mobx';
import validate from 'mobx-form-validate';
import Session from '../../../logics/session';

export default class LoginForm {
  @observable
  @validate(/^1\d{10}$/, 'Please enter a valid mobile')
  mobile = '';

  @observable
  @validate(/^.+$/, 'Please enter any password')
  pwd = '';

  // Bind this for future use.
  submit = () => {
    return Session.login(this, 'user');
  }
}

const form = new LoginForm();
console.log(form.validateErrorMobile);  // Please enter a valid mobile
console.log(form.validateError);        // Please enter a valid mobile
console.log(form.isValid);              // false

```

### Use with react
 
```js
import React from 'react';
import { observer } from 'mobx-react'; 
import LoginForm from './LoginForm';

@observer
export default class Login extends React.Component {
  form = new LoginForm();
  render() {
    return (
      <div>
        <p>Mob: <input value={this.form.mobile} onChange={ev=>this.form.mobile = ev.target.value}/></p>
        <p>{this.form.validateErrorMobile}</p>
        <p>Pwd: <input type="password" value={this.form.pwd} onChange={ev=>this.form.pwd = ev.target.value}/></p>
        <p>{this.form.validateErrorPwd}</p>
        <button disabled={!this.form.isValid} onClick={this.form.submit}>Submit</button>
      </div>
    )
  }
}
```

### Use with react-native

Just replace all html element with react-native components.

```js
import React from 'react';
import { observer } from 'mobx-react';
import { View, Text, TextInput, TouchableOpacity } from 'react-native';
import LoginForm from './LoginForm';

@observer
export default class Login extends React.Component {
  form = new LoginForm();
  render() {
    return (
      <View>
        <View><Text>Mob:</Text> <TextInput value={this.form.mobile} onChangeText={text=>this.form.mobile = text}/></View>
        <View>{this.form.validateErrorMobile}</View>
        <View><Text>Pwd:</Text> <TextInput type="password" value={this.form.pwd} onChangeText={text=>this.form.pwd = text}/></View>
        <View>{this.form.validateErrorPwd}</View>
        <TouchableOpacity disabled={!this.form.isValid} onPress={this.form.submit}><Text>Submit</Text></button>
      </View>
    )
  }
}
```

### Custom valid condition

You can define your own `isValid` getter, with any additional condition:

```js
class MyForm {
  startAt = new Date();
  
  @computed
  get isValid() {
    // This form is only submittable after 10 seconds.
    return !this.validateError && new Date() - this.startAt > 10000; 
  ]
}
```

### Optimize

To avoid re-render of the whole form, you can create a item component to observe
 a single field:

```js
import React from 'react';
import { observer } from 'mobx-react'; 
import LoginForm from './LoginForm';
import camelCase from 'camelcase';

// Only re-render when this field changes.
const Input = observer(function Input({label, form, name, ...others}){
  return (
    <p>
        <input value={form[name]} onChange={ev=>form[name]=ev.target.value} {...others}/>
        {form[camelCase('validateError', name)]}
    </p>
  )
});

// Only re-render when the whole form become valid or invalid.
const Submit = observer(function Submit({form, children}){
  return <button disabled={!form.isValid} onClick={this.form.submit}>{children}</button>
});

// Do not re-render.
export default class Login extends React.Component {
  form = new LoginForm();
  render() {
    return (
      <div>
        <Input label="Mob:" name="mobile" form={this.form} />
        <Input label="Pwd:" name="pwd" form={this.form} type="password"/>
        <Submit form={this.form}>Submit</Submit>
      </div>
    )
  }
}
```

### Sub-form

You can create sub-form or even sub-form list in your form:

```js
class SubForm {
  @observable
  @validate(/.+/)
  field1 = '';
  
  @observable
  @validate(/.+/)
  field2 = '';
}

class Item {
  @observable
  @validate(/.+/)
  field3 = '';
}

class MainForm {
  @observable
  haveSubForm = false;
    
  @observable
  @validate((value, this)=>this.haveSubForm && value.validationError)
  subForm = new SubForm();
  
  @observable
  @validate((value) => value.map(v=>v.validationError).find(v=>v))
  itemList = [];
  
  @action
  addItem() {
    this.itemList.push(new Item());
  }
  
  @action
  clearItem() {
    this.itemList.clear();
  }
}
```

### Array of req & msg

#一般用法：

```js
@observable
@validate([/^.+$/, /^[abc]+$/, /^[abc]{3}$/], ['请输入密码', '只能有abc', '必须是3位字符'])
password = '';
```

#req 数组中带有 function 的用法：

用法1：function 在 req 数组中间（非最后一个），则返回 `true`/`false`，
错误提示为 msg 数组中对应的字符串。

```js
@observable
  @validate([/^.+$/, (value) => {return value.indexOf('pizza') !== -1 ? true : false;}, /.{6,}/], ['请输入用户名或手机号', '必须包含字符串 pizza', '必须大于6位'])
userName = '';
```

用法2: function 为 req 数组中最后一个，则返回 `undifined` 表示通过验证，
返回字符串则表示验证不通过，且作为错误提示。`此用法可用于不同情况返回不同提示的应用场景`

```js
@observable
@validate([/^.+$/, (value) => {return value == 'pizza' ? undefined : '请输入 pizza';}], ['请输入用户名或手机号'])
userName = '';
```

> 如果是判断逻辑很复杂的情况，建议使用单个 function 解决，使用方法为 `@validate(func)` , `func` 为一个验证方法，返回`undefined`表示验证通过，否则返回提示字符串。


#特殊用法：

有时我们的项目中表单是根据业务逻辑来显示或隐藏元素的，只有在表单元素显示的情况下才会做出验证(如：登录时，只用用户输入错误了用户名或密码3次以上才出现图形验证码)，此时就可以先把表单元素对应的@observable属性为设置null，在表单显示元素显示时，再给他设置一个默认值（如this.verifyCode=''）即可

```js
@observable
@validate([/^.+$/], ['请输入验证码'])
verifyCode = null;
```
