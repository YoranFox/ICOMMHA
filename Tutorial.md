## nestjs auto dto conversion with interceptor
This is a technique that is used to convert a entity to a defined dto in the NestJS framework. 

### Packages used
- class-transformer https://github.com/typestack/class-transformer
- class-validator


### How to install
```
npm install --save class-transformer
npm install --save class-validator
```

### Transformer interceptor
This is the interceptor that is used to transform the entities. The function painToClass is being used to transform the classes. 

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

### Entity
This is the database entity where all the values are defined that can be used by the DTO and are saved in the database. Since SQL is used, we use the TypeORM module for defining what the columns look like.

A great trick is to use a getter as a new attribute to use the variables in the entity to create new data. See the last attribute. This getter can be used in the dto's.

```
@Entity()
export class User extends BaseEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ nullable: false })
  googleId: string;

  @Column({ nullable: false })
  givenName: string;

  @Column({ nullable: false })
  familyName: string;

  @Column({ nullable: false })
  email: string;

  @Column({ default: () => true })
  active: boolean;

  @ManyToMany(() => Role, { eager: true, cascade: true })
  @JoinTable()
  roles: Role[];

  @Expose()
  get name() {
    return this.givenName + ' ' + this.familyName;
  }
}
```

### DTO
This is the dto that the entity will be transformed into, the decorators that are being used are from class-transformer. Here the @Expose() tells the transformer to keep the attribute. If a attribute is of a different type the @Type() decorator has to be used. This can be used in combination with other dto's that use the same layout as this one to transform a typed object into a dto. 

```
@Exclude()
export class ResponseDetailUserDto {
  @Expose()
  @IsNumber()
  id: number;
  @Expose()
  @IsString()
  givenName: string;
  @Expose()
  @IsString()
  familyName: string;
  @Expose()
  @IsEmail()
  email: string;
  @Expose()
  picture: string;
  @Type(() => Role)
  @Expose()
  roles: Role[];
  @Expose()
  active: boolean;
  @Expose()
  name: string;
}
```

### Controller
To send the correct data that is transformed to the dto, we use the interceptor on the endpoint as a decorator. Here we give the DTO type to the interceptor. 

In the example the returned user is of type User. 

```
  @Get('/profile')
  @UseInterceptors(new TransformInterceptor(ResponseDetailUserDto))
  async getProfile(@Req() req): Promise<ResponseDetailUserDto> {
    return req.user;
  } 
```
