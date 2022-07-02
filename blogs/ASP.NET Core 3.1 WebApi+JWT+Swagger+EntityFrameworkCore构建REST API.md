# 一、准备
- 使用vs2019新建ASP.NET Core Web应用程序，选用api模板：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520102936937-903912270.png)
- 安装相关的NuGet包：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520103408177-1092131134.png)
# 二、编码
- 首先编写数据库模型：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520104621913-78830182.png)
用户表 User.cs:
```
public class User
    {
        [Key]
        public Guid ID { get; set; }

        [Required]
        [Column(TypeName = "VARCHAR(16)")]
        public string UserName { get; set; }

        [Required]
        [Column(TypeName = "VARCHAR(16)")]
        public string Password { get; set; }
    }
```
数据库上下文 DemoContext.cs，在数据库创建时增加一条种子数据admin：
```
public class DemoContext : DbContext
    {
        public DemoContext(DbContextOptions<DemoContext> options)
            : base(options)
        {

        }

        public DbSet<User> Users { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            modelBuilder.Entity<User>().HasData(new User
            {
                ID = Guid.Parse("94430DDF-E6E1-4836-A7D2-49A9FCEF722E"),
                UserName = "admin",
                Password = "123456"
            });
        }
    }
```
- 编写数据访问服务：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520105343102-2038985088.png)
IUserService接口，这里简单定义几个添加查询的方法：
```
public interface IUserService
    {
        Task<IEnumerable<User>> GetUserAsync();

        Task<User> GetUserAsync(Guid id);

        Task<User> GetUserAsync(string username, string password);

        Task<User> AddUserAsync(string username, string password);
    }
```
UserService实现类：
```
public class UserService : IUserService
    {
        private readonly DemoContext context;

        public UserService(DemoContext context)
        {
            this.context = context ?? throw new ArgumentNullException(nameof(context));
        }

        public async Task<User> AddUserAsync(string username, string password)
        {
            User user = new User();
            user.ID = Guid.NewGuid();
            user.UserName = username;
            user.Password = password;
            await context.Users.AddAsync(user);
            context.SaveChanges();
            return user;
        }

        public async Task<User> GetUserAsync(string username, string password)
        {
            return await context.Users.FirstOrDefaultAsync(p => p.UserName == username && p.Password == password);
        }

        public async Task<IEnumerable<User>> GetUserAsync()
        {
            return await context.Users.ToListAsync();
        }

        public async Task<User> GetUserAsync(Guid id)
        {
            return await context.Users.FirstOrDefaultAsync(p => p.ID == id);
        }

    }
```
- appsettings.json中增加jwt，efcore相关的配置 JwtSetting、ConnectionStrings：
```
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "JwtSetting": {
    "SecurityKey": "88d082e6-5672-4c6c-bc42-6fcce20fbf51", // 密钥
    "Issuer": "jwtIssuertest", // 颁发者
    "Audience": "jwtAudiencetest", // 接收者
    "ExpireSeconds": 3600 // 过期时间（3600）
  },
  "ConnectionStrings": {
    "DemoContext": "data source=.;Initial Catalog=WebApiDemoDB;User ID=sa;Password=123456;MultipleActiveResultSets=True;App=EntityFramework"
  }
}
```
- 增加jwt配置对象：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520110543157-1320523038.png)
```
    /// <summary>
    /// jwt配置对象
    /// </summary>
    public class JwtSetting
    {
        public string SecurityKey { get; set; }
        public string Issuer { get; set; }
        public string Audience { get; set; }
        public int ExpireSeconds { get; set; }
    }
```
```
public static class AppSettings
    {
        public static JwtSetting JwtSetting { get; set; }

        /// <summary>
        /// 初始化jwt配置
        /// </summary>
        /// <param name="configuration"></param>
        public static void Init(IConfiguration configuration)
        {
            JwtSetting = new JwtSetting();
            configuration.Bind("JwtSetting", JwtSetting);
        }
    }
```
- 在Startup.cs中配置相关服务和中间件：
```
public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            AppSettings.Init(Configuration);

            services.AddSwaggerGen(c =>
            {
                c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
                // Set the comments path for the Swagger JSON and UI.
                var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
                var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
                c.IncludeXmlComments(xmlPath);
                c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme()
                {
                    Description = "在下框中输入请求头中需要添加Jwt授权Token：Bearer Token",
                    Name = "Authorization",
                    In = ParameterLocation.Header,
                    Type = SecuritySchemeType.ApiKey,
                    BearerFormat = "JWT",
                    Scheme = "Bearer"
                });

                c.AddSecurityRequirement(new OpenApiSecurityRequirement
                    {
                        {
                            new OpenApiSecurityScheme{
                                Reference = new OpenApiReference {
                                            Type = ReferenceType.SecurityScheme,
                                            Id = "Bearer"}
                           },new string[] { }
                        }
                    });
            });

            services
              .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
              .AddJwtBearer(options =>
              {
                  options.TokenValidationParameters = new TokenValidationParameters
                  {
                      ValidIssuer = AppSettings.JwtSetting.Issuer,
                      ValidAudience = AppSettings.JwtSetting.Audience,
                      IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(AppSettings.JwtSetting.SecurityKey)),
                      // 默认允许 300s  的时间偏移量，设置为0
                      ClockSkew = TimeSpan.Zero,
                  };
              });

            services.AddCors(options =>
            {
                options.AddPolicy("any",
                    builder =>
                    {
                        builder.AllowAnyMethod()
                            .AllowAnyOrigin()
                            .AllowAnyHeader();
                    });
            });

            services.AddControllers();
            services.AddScoped<IUserService, UserService>();
            services.AddDbContext<DemoContext>(opt => opt.UseSqlServer(Configuration.GetConnectionString("DemoContext")));
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            app.UseAuthentication();

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseSwagger();
            app.UseSwaggerUI(c =>
            {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
            });

            app.UseRouting();

            app.UseAuthorization();

            //CORS 中间件必须配置为在对 UseRouting 和 UseEndpoints的调用之间执行。 配置不正确将导致中间件停止正常运行。
            app.UseCors("any");

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }
```
- 打开项目文件，增加项目xml文档生成配置，swagger需要用到：
```
<GenerateDocumentationFile>true</GenerateDocumentationFile>
<NoWarn>$(NoWarn);1591</NoWarn>
```
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520111114679-1796576398.png)
- 数据库迁移：
打开程序包管理控制台：执行命令`Add-Migration Initial`
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520111544194-1547226542.png)
然后执行`Update-Database`
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520111924634-1364584001.png)
此时数据库已经成功生成：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520112100696-1543727758.png)
- 下面是controller：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520112238372-1895523819.png)
先建一个数据传输实体，方便统一controller的返回值：
```
public class BaseDto<T>
    {
        public BaseDto(StatusCode code, string message)
        {
            Code = code;
            Message = message;
        }

        public BaseDto(StatusCode code, string message, T data)
        {
            Code = code;
            Message = message;
            Data = data;
        }

        public StatusCode Code { get; set; }

        public string Message { get; set; }

        public T Data { get; set; }
    }

    public enum StatusCode
    {
        Success = 0,
        Error = 1,
    }
```
UserController:
```
/// <summary>
    /// 用户
    /// </summary>
    [Authorize]
    [Route("api/[controller]")]
    [ApiController]
    public class UserController : ControllerBase
    {
        private readonly IUserService userService;

        public UserController(IUserService userService)
        {
            this.userService = userService;
        }

        /// <summary>
        /// 所有用户
        /// </summary>
        /// <returns></returns>
        [Route("")]
        [HttpGet]
        public async Task<ActionResult<BaseDto<IEnumerable<User>>>> Get()
        {
            var users = await userService.GetUserAsync();
            BaseDto<IEnumerable<User>> dto = new BaseDto<IEnumerable<User>>(Dto.StatusCode.Success, "", users);
            return Ok(dto);
        }

        /// <summary>
        /// 当前用户
        /// </summary>
        /// <returns></returns>
        [Route("me")]
        [HttpGet]
        public async Task<ActionResult<BaseDto<User>>> UserInfo()
        {
            string id = User.FindFirst("id")?.Value;
            var user = await userService.GetUserAsync(Guid.Parse(id));
            BaseDto<User> dto = new BaseDto<User>(Dto.StatusCode.Success, "", user);
            return Ok(dto);
        }

        /// <summary>
        /// 根据ID获取用户
        /// </summary>
        /// <param name="id"></param>
        /// <returns></returns>
        [Route("{id}")]
        [HttpGet]
        public async Task<ActionResult<BaseDto<User>>> Get(Guid id)
        {
            var user = await userService.GetUserAsync(id);
            BaseDto<User> dto = new BaseDto<User>(Dto.StatusCode.Success, "", user);
            return Ok(dto);
        }

        /// <summary>
        /// 添加用户
        /// </summary>
        /// <param name="loginParameter"></param>
        /// <returns></returns>
        [HttpPost]
        public async Task<ActionResult<BaseDto<User>>> Add(LoginParameter loginParameter)
        {
            var user = await userService.AddUserAsync(loginParameter.UserName, loginParameter.Password);
            BaseDto<User> dto = new BaseDto<User>(Dto.StatusCode.Success, "", user);
            return Ok(dto);
        }
    }

    public class LoginParameter
    {
        public string UserName { get; set; }

        public string Password { get; set; }
    }
```
TokenController:
```
/// <summary>
    /// 鉴权
    /// </summary>
    [Route("api/[controller]")]
    [ApiController]
    public class TokenController : ControllerBase
    {
        private readonly IUserService userService;

        public TokenController(IUserService userService)
        {
            this.userService = userService;
        }

        /// <summary>
        /// 获取token
        /// </summary>
        /// <param name="loginParameter"></param>
        /// <returns></returns>
        [AllowAnonymous]
        [HttpPost(Name = nameof(Login))]
        public async Task<ActionResult<BaseDto<object>>> Login([FromBody]LoginParameter loginParameter)
        {
            var user = await userService.GetUserAsync(loginParameter.UserName, loginParameter.Password);
            if (user != null)
            {
                var token = AppHelper.Instance.GetToken(user);
                BaseDto<object> dto = new BaseDto<object>(Dto.StatusCode.Success, "", new { token });
                return Ok(dto);
            }
            return Ok(new BaseDto<object>(Dto.StatusCode.Error, "", null));
        }
    }
```
AppHelper中生成token的方法：
```
public class AppHelper
    {
        public readonly static AppHelper Instance = new AppHelper();

        private AppHelper() { }

        /// <summary>
        /// 生成token
        /// </summary>
        /// <param name="user"></param>
        /// <returns></returns>
        public string GetToken(User user)
        {
            //创建用户身份标识，可按需要添加更多信息
            var claims = new Claim[]
            {
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
                new Claim("id", user.ID.ToString(), ClaimValueTypes.Integer32), // 用户id
                new Claim("name", user.UserName), // 用户名
            };

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(AppSettings.JwtSetting.SecurityKey));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

            //创建令牌
            var token = new JwtSecurityToken(
              issuer: AppSettings.JwtSetting.Issuer,
              audience: AppSettings.JwtSetting.Audience,
              signingCredentials: creds,
              claims: claims,
              notBefore: DateTime.Now,
              expires: DateTime.Now.AddSeconds(AppSettings.JwtSetting.ExpireSeconds)
            );

            string jwtToken = new JwtSecurityTokenHandler().WriteToken(token);

            return jwtToken;
        }

    }
```
# 三、效果
运行项目，浏览器访问：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520113051099-1086309716.png)
测试一下用户接口：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520113205089-2120476123.png)
这时返回401错误，因为我们还没有鉴权
使用admin/123456获取token：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520113545753-465851037.png)
拿到token 点击authorize：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520113653764-1982033808.png)
然后再测试用户接口：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200520113957244-118696772.png)
此时已经可以正常请求。
代码：https://github.com/xiajingren/NetCore3.1-WebApi-Demo