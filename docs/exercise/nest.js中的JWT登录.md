# JWT认证
最近在看nest.js，之前都在接触session这种认证方式，这次记录一下jwt这块。
## JWT是什么
> JSON Web Token（以下简称 JWT）是一套开放的标准，它定义了一套简洁（compact）且 URL 安全（URL-safe）的方案，以安全地在客户端和服务器之间传输 JSON 格式的信息。

区别于常用的session & cookie的认证方式，这种认证机制也是一种**无状态**的，server不需要保存类似于session的认证信息，只要客户端保存好token。

// TODO 待完善 先占个坑

## nest.js中如何使用

在nest.js中使用认证，可以先阅读一下[官方文档的authentication](https://docs.nestjs.com/security/authentication)，文档以及说的很详细了。
主要是使用了passport作为auth的依赖库，并结合**local策略 + JWT策略**。local策略即用户名/密码认证机制。本文是按照文档上，用户先通过用户名/密码登录，认证成功后生成JWT并作为后续认证的凭证。

### 各种依赖
这里主要使用到了如下依赖:
1. node的一个比较成熟的认证库passwport以及@nestjs/passport、本地策略passport-local。
```bash
$ npm install --save @nestjs/passport passport passport-local
$ npm install --save-dev @types/passport-local
```
2. JWT策略的依赖
```bash
$ npm install --save @nestjs/jwt passport-jwt
$ npm install --save-dev @types/passport-jwt
```

### 认证过程
按我的理解画了个认证的步骤：

![在线制图-](https://image-1254278777.cos.ap-guangzhou.myqcloud.com/blog/在线制图-.png)

### code
按照上面的认证步骤：
1. 用户注册，这里只简单写了controller，具体的新用户、用户名校验等可以后面通过class-validator校验。
```js
// users.controller.ts
@Post('/register')
async register (@Body() user:User) {
  // TODO: 校验
  return await this.usersService.create(user)
}
```

2. 添加login路由
```js
// auth.controller.ts
@Post('/login')
async login(@Body() user:Pick<User, 'username'|'password'>) {
  return this.authService.login(user as User)
}
```

3. 实现authService
auth的service主要完成2个方法：
(1). local策略时对password的验证
(2). local策略 登录后下发token
```js
@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService
  ) {}

  async validate (username: string, password: string) {
    // TODO
  }

  async login (user: User) {
    const payload = { username: user.username, id: user._id };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

4. 实现认证策略
4.1 local
```js
// local.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common'
import { PassportStrategy } from '@nestjs/passport'
import { Strategy } from 'passport-local' // 这里不要引用错了
import { AuthService } from './auth.service'

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super()
  }

  // 要实现一个validate方法
  async validate (username: string, password: string) {
    const user = await this.authService.validate(username, password)
    if (user) {
      return user
    } else {
      throw new UnauthorizedException()
    }
  }
}
```

4.2 jwt
```js
// jwt.strategy.ts
import { Injectable } from '@nestjs/common'
import { PassportStrategy } from '@nestjs/passport'
import { JwtConfig } from 'apps/client/utils/constants'
import { ExtractJwt, Strategy } from 'passport-jwt'

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    // jwt策略需要一些初始化options
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: 'secretKey'
    })
  }
  // Passport首先验证JWT的签名并解码JSON,然后,它调用我们的validate方法，将解码后的JSON作为其单个参数传递。并且，基于validate方法的返回值构建一个用户对象，并将其作为属性附加到Request对象上。
  async validate(payload: any) {
    return { id: payload.id, username: payload.username };
  }
}
```

5. 注册passport的策略
```js
// auth.module.ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule} from '../users/users.module'
import { JwtConfig } from '../../utils/constants'
import { JwtModule } from '@nestjs/jwt'
import { PassportModule } from '@nestjs/passport'
import { AuthController } from './auth.controller'
import { LocalStrategy } from './local.strategy '
import { JwtStrategy } from './jwt.strategy'

@Module({
  imports: [
    UsersModule,
    // 这里引入Module
    JwtModule.register(JwtConfig),
    PassportModule
  ],
  // 这里注册策略
  providers: [AuthService, LocalStrategy, JwtStrategy],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

6. 使用策略
使用Guard作为路由的前置来实现校验。
可以使用@nestjs/passport的内置的Guard

当未经身份验证的用户尝试登录时
```js
// auth.controller.ts
// 这里使用的就是内置Guard, 指明用哪种策略
@UseGuards(AuthGuard('local'))
@Post('/login')
async login(@Body() user:Pick<User, 'username'|'password'>) {
  return this.authService.login(user as User)
}
```

当已经有身份验证的用户尝试请求需要登录的路由时，使用```@UseGuards(AuthGuard('jwt'))```

## api验证
使用了@nest/swagger作为api的一个doc，也可以很方便的进行api测试。
对于需要jwt的api，在main.ts入口文件处，添加addBearerAuth，就可以输入登录后下发的token了。
```js
new DocumentBuilder().addBearerAuth()
```
并在控制器路由前使用```ApiBearerAuth()```装饰器。

### 登录
![20201227215810](https://image-1254278777.cos.ap-guangzhou.myqcloud.com/blog/20201227215810.png)

### 没有token时请求需要auth的api
![20201227215911](https://image-1254278777.cos.ap-guangzhou.myqcloud.com/blog/20201227215911.png)

### 输入Token再请求
![20201227220033](https://image-1254278777.cos.ap-guangzhou.myqcloud.com/blog/20201227220033.png)

参考：
[Nest.js 从零到壹系列（三）：使用 JWT 实现注册、登录](https://juejin.cn/post/6844904097317912584)
[NNC D9：在 Nest.js 应用里验证用户的身份（基于 JWT）](https://ninghao.net/blog/7323)
[nest.js authentication](https://docs.nestjs.com/security/authentication)