# Laravel实践中遇到的一些坑

1. `mpociot/laravel-apidoc-generator`由于注释星号个数问题导致生成的接口文档缺失。正确格式为 `/** xxxx */`, 开头的星号只能为2个

2. `mpociot/laravel-apidoc-generator`运行文档生成命令时碰到下面的错误:

   ```
   In SKUserProductController.php line 216:
   
   Unparenthesized `a ? b : c ? d : e` is deprecated. Use either `(a ? b : c) ? d : e` or `a ? b : (c ? d : e)`
   ```

   原因是PHP7.4不支持了三元表达式的一些写法，更正对应的代码块或者用低版本的PHP运行。

   