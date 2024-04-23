# NestJS auto DTO conversion with interceptor
In this tutorial we will be discussing the possibilities and technical aspects of a auto DTO transformer when using NestJS. Once you start using NestJS you will find that using DTO's for your enteties and data objects is a important step. We will be taking this even one step further by allowing you to define your DTO's and transform them without having to program a extra transformation step for each endpoint.

## Content
* Prerequisite
* Why use a DTO
* Automatic transformation
* Conclusion with extra examples

## Prerequisite

### Packages used
- class-transformer https://github.com/typestack/class-transformer
- class-validator

### How to install
```
npm install --save class-transformer
npm install --save class-validator
```

## Why do we use DTO's
When sending back data from our API we need to make sure that the data we send back does not contain unauthenticated/sensitive data and is the data that is needed. This is done by transforming the entities to a Data Tranfer Object namely a DTO. In this tutorial we will be using a User entity as a example.

```
@Entity()
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ nullable: false })
  givenName: string;

  @Column({ nullable: false })
  familyName: string;

  @Column({ nullable: false })
  email: string;

  @Column({ nullable: false })
  password: string;

  @Column({ default: () => true })
  active: boolean;

  @ManyToMany(() => Role, { eager: true, cascade: true })
  @JoinTable()
  roles: Role[];
}
```

When we have a application that wants to display a user we dont want to sent back sensitive data like a password or roles. This is where we need to define a DTO to determine what kind of data is send when a endpoint is requested.

```
@Exclude()
export class ResponseDisplayUserDto {
  @Expose()
  @IsString()
  id: string;
  @Expose()
  @IsString()
  givenName: string;
  @Expose()
  @IsString()
  familyName: string;
  @Expose()
  @IsEmail()
  email: string;
}
```
We removed the roles and password fields and kept the fields we want to send back. This is only the first step and doesn't actually transforms anything yet and only gives us a Type that we can use to define a data object.

```
  @Get('/:id/display')
  async getDisplay(@Param('id') id: string): Promise<ResponseDisplayUserDto> {
    const user: User = this.userService.find(id);

    const displayUser: ResponseDisplayUserDto = {
      id: user.id,
      givenName: user.givenName,
      familyName: user.familyName,
      email: user.email
    } 
    
    return displayUser;
  } 
```

The fields are mapped to the right fields of the DTO and is returned. The only downside is that you need to do this for every endpoint that you make. This alone is already a usefull tool but we want to make sure this transformation is done automatically.


## Automatic transformation
To transform a entity to a DTO described above we need to utilize the `plainToClass` function provided by the class-transformer package. Here we can send the DTO to transform to and the data to transform. But to be able to do this for every request we need to create a Interceptor in NestJS and apply this logic.

```
interface ClassType<T> {
  new (): T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<Partial<T>, T> {
  constructor(private readonly classType: ClassType<T>) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<T> {
    return next
      .handle()
      .pipe(map((data) => plainToClass(this.classType, data)));
  }
}
```

So now we can use this interceptor and apply it to the endpoint and give in the DTO class and the data object. Now we can skipp the manual tranformation step and just return the data object (User entity).

```
  @Get('/:id/display')
  @UseInterceptors(new TransformInterceptor(ResponseDisplayUserDto))
  async getDisplay(@Param('id') id: string): Promise<ResponseDisplayUserDto> {
    const user: User = this.userService.find(id);
    return user;
  } 
```


## Conclusion with extra examples

The above solution gives us the tools to automate the work flow of tranforming and returning the right data objects for a endpoint. There are a few handy features that you can use when applying this.

### Related Entities and DTO's
When requesting a Entity there can also be a related entity that has to be transformed to a DTO. How can we define this and use it in the solution discussed above. It is actually simpler than you would think and can be done by simply adding a DTO as the variable type in the parent DTO. Let's take the User DTO as an example. We want to create a request where the user is returned with a readable version of the roles.

```
@Entity()
export class Role {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ nullable: false })
  role: string;

  @Column({ nullable: false })  
  accessLevel: number;

  get readableRole(): string {
    return RoleHelper.getReadableRole(this.role);
  }
}
```

```
@Exclude()
export class ResponseReadableRoleDto {
  @Expose()
  @IsString()
  id: string;
  @Expose()
  @IsString()
  readableRole: string;
}
```

```
@Exclude()
export class ResponseUserWithReadableRolesDto {
  @Expose()
  @IsString()
  id: string;
  @Expose()
  @IsString()
  givenName: string;
  @Expose()
  @IsString()
  familyName: string;
  @Expose()
  @Type(() => ResponseReadableRoleDto)
  roles: ResponseRoleDto;
}
```

```
  @Get('/:id/with-readable-roles')
  @UseInterceptors(new TransformInterceptor(ResponseUserWithReadableRolesDto))
  async getUserWithReadableRoles(@Param('id') id: string): Promise<ResponseUserWithReadableRolesDto> {
    const user: User = this.userService.find(id);
    return user;
  } 
```

We can see that the only thing that changed in the endpoint is the new DTO type that is given to the tranformer and the return type. Also a thing to note is the getter that is used in the DTO to transform the data of the entity to a different format.  

