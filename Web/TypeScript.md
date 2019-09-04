### Basic Types

- 基本类型：string, number, boolean undefined, null
- 变量声明：`count: number`, `name: string`
- 函数声明：`function add(left: number, right: number): number`
- any：无特定类型，一般来源于三方库，应该少用，无类型检查
- unknown：检查类型，替换掉any
- void：返回值类型，无返回
- never：不应该发生的类型，一般写在不可到达的代码
- enum：类似Java or C++
- object：复杂类型

#### Collections

- array：`const numbers: number[] = [];`
    - `for (let i in numbers) {}`
    - `numbers.forEach(function (num) {});`
- tuple：类似数组，但元素个数固定，而且可以有不同类型元素
- `...`：rest和spread语法
- 通过rest语法，可以实现开放结尾的元组`type Scores = [string, ...number[]];`
- 可以有空元组`type Empty = [];`

### Classes

- `interface A {}`
- 接口：属性和方法声明，不能有方法实现
- 可选参数和属性，通过加?到变量名后面
- `a || 0` means if a is false, 0 is used
- false, 0, "", null, undefined, NaN -> are false
- 只读属性`readonly name: string`初始化以后不可更改
- 类型别名：`type GetTotal = (discount: number) => number;`
- 类型别名和接口类似，但不能继承
- `class`用来声明类型，`const product = new Product()`
- `class Product implements IBrand`实现接口
- 构造方法：`constructor(public name: string, size: number = 2)`
- 扩展用`extends`
- 子类用`super`调用基类方法
- `abstract class Product`抽象类不能实例化
- 抽象类的抽象方法子类必须实现
- 所有属性和方法默认public，也可以使用`private`,`protected`.
- getter and setter用来实现属性的读写控制
```
  private _unitPrice: number;
  get unitPrice(): number {
    return this._unitPrice || 0;
  }
  set unitPrice(value: number) {
    if (value < 0) {
      value = 0;
    }
    this._unitPrice = value;
  }
```
- static methods and props like Java etc.
- 类型推断：`a is A`
- 属性检查：`"name" in A`
- 类型转换：`a as B`

### Modules

- 默认包是global
- AMD、CMD、UMD、CommonJS(require, module.exports)、ES6(import,export)
- `export interface Product {}`
- `export { Product }` `export { Product as A }`
- `import { Product as A } from "./product"`
- `export default interface { name: string; }`  `import Product from "./product"`只能有一个default

### Compile

- `tsc a.ts`
- `tsc a.ts --target es6`, default ES3
- `tsc a.ts --outDir dist`, default same dir of src file
- `tsc a.ts --module es6`, default commonjs
- `tsc b.js --allowJS`, allow transpile JS to different target
- `tsc a.ts --watch`, auto compile when file changes
- `tsc a.ts --noImplicitAny`, force to specify type `any` in code
- `tsc a.ts --noImplicitReturns`, force return from method if get a return type
- `tsc a.ts --sourceMap`, generate \*.map for development to debug ts
- `tsc a.ts --moduleResolution node`, 使用的包管理方式npm，classic for ES6
- `tsc --build project`，增量编译，`--force`强制全量编译

#### tsconfig.json

- ts项目配置 ，编译配置`compilerOptions` 

- 多项目结构需要配置composite，但同时要设置declaration（生成.d.ts）：
  ```json
  {
    "compilerOptions": {
      "composite": true,
      "declaration": true,
      ...
    }
  }
  ```

- `files`：编译的代码文件

- `include`：编译代码目录

- `exclude`：不被编译代码目录
- `declarationMap`：设置为true，可以用于调转到定义

#### tslint.json

- tslint例子
```json
    {
        "extends": ["tslint:recommended"],
        "rules": {
            "member-access": true,
            "interface-name": false
        },
        "linterOptions": {
            "exclude": ["node_modules/**/*.ts"]
        },
        "references": [
        	  {"path": "../shared"}
        ]
    }
```
> extends：扩展/继承默认设置
> rules：自定义规则，覆盖默认
> linterOptions：lint设置
> references：依赖设置

