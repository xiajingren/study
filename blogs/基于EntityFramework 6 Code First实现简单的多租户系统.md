# 一、前言

公司原本有一个“xx系统”，ORM使用EntityFramework，Code First模式。该系统是针对某个客户企业的，现要求该系统支持多个企业使用，但是又不能给每个企业部署一份（难以维护），只能想办法从代码层面去解决这个问题。

# 二、思路

1. 在原有的数据表增加外键，标记该数据属于哪个企业。这代码改动会非常大，之前的查询修改代码都需要增加外键筛选的逻辑。这显然不合理。
2. 动态分库。每个企业注册时，为他生成一个独立的数据库，企业登录时切换到他对应的数据库。这样就完全不用修改以前的业务代码，只需要考虑企业数据库切换的问题。

# 三、实现

那么EntityFramework Code First模式怎么实现动态分库的功能呢？
1. 首先建立一个主库，主库只存放企业用户的数据，包括企业登录名，密码，对应的数据库名 等等...  主库只有一个。
2. 业务数据库，在企业注册的时候动态创建，业务数据库可以有多个，也可以放到不同的服务器。
3. 企业登录时，读取主库，拿到业务数据库名称，然后保存到用户session中（也可以是别的缓存），该用户的后续请求都基于此数据库。

为了简单我建立了一个demo项目：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518150449775-1078319520.png)

主库模型放在XHZNL.EFDynamicDatabaseBuilding.MasterEntity里面，主库只有一个企业表：Enterprise:
```
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace XHZNL.EFDynamicDatabaseBuilding.MasterEntity
{
    /// <summary>
    /// 企业
    /// </summary>
    public class Enterprise
    {
        /// <summary>
        /// ID
        /// </summary>
        [Required]
        public Guid ID { get; set; }

        /// <summary>
        /// 企业名称
        /// </summary>
        [Required]
        [Column(TypeName = "NVARCHAR")]
        [MaxLength(50)]
        public string Name { get; set; }

        /// <summary>
        /// 企业数据库名称
        /// </summary>
        [Required]
        [Column(TypeName = "NVARCHAR")]
        [MaxLength(100)]
        public string DBName { get; set; }

        /// <summary>
        /// 企业 账号
        /// </summary>
        [Required]
        [Column(TypeName = "NVARCHAR")]
        [MaxLength(20)]
        public string AdminAccount { get; set; }

        /// <summary>
        /// 企业 密码
        /// </summary>
        [Required]
        [Column(TypeName = "NVARCHAR")]
        [MaxLength(50)]
        public string AdminPassword { get; set; }
    }
}
```
XHZNL.EFDynamicDatabaseBuilding.MasterEntity.Services.BaseService:
```
using System;
using System.Collections.Generic;
using System.Data.Entity;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using XHZNL.EFDynamicDatabaseBuilding.Common;

namespace XHZNL.EFDynamicDatabaseBuilding.MasterEntity.Services
{
    public class BaseService
    {
        /// <summary>
        /// 获取context
        /// </summary>
        /// <returns></returns>
        internal MasterDBContext GetDBContext()
        {
            try
            {
                var context = new MasterDBContext();

                if (!context.Database.Exists())
                {
                    context.Database.Create();

                    var dbInitializer = new MigrateDatabaseToLatestVersion<MasterDBContext, Migrations.Configuration>(true);
                    dbInitializer.InitializeDatabase(context);
                }

                if (!context.Database.CompatibleWithModel(false))
                {
                    var dbInitializer = new MigrateDatabaseToLatestVersion<MasterDBContext, Migrations.Configuration>(true);
                    dbInitializer.InitializeDatabase(context);
                }

                return context;
            }
            catch (Exception ex)
            {
                return null;
            }
        }
    }
}
```
XHZNL.EFDynamicDatabaseBuilding.MasterEntity.Services.EnterpriseService:
```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using XHZNL.EFDynamicDatabaseBuilding.Common;

namespace XHZNL.EFDynamicDatabaseBuilding.MasterEntity.Services
{
    /// <summary>
    /// 企业服务
    /// </summary>
    public class EnterpriseService : BaseService
    {
        public static readonly EnterpriseService Instance = new EnterpriseService();

        private EnterpriseService() { }

        /// <summary>
        /// 根据账号密码 获取 企业
        /// </summary>
        /// <param name="account"></param>
        /// <param name="password"></param>
        /// <returns></returns>
        public Enterprise Get(string account, string password)
        {
            try
            {
                using (var context = GetDBContext())
                {
                    var model = context.Enterprises.FirstOrDefault(m => m.AdminAccount == account && m.AdminPassword == password);
                    if (model != null)
                    {
                        //设置当前业务数据库
                        CommonHelper.Instance.SetCurrentDBName(model.DBName);
                    }
                    return model;
                }
            }
            catch (Exception ex)
            {
                return null;
            }
        }

        /// <summary>
        /// 添加企业
        /// </summary>
        /// <param name="id"></param>
        /// <returns></returns>
        public bool Add(Enterprise enterprise)
        {
            try
            {
                using (var context = GetDBContext())
                {
                    enterprise.ID = Guid.NewGuid();
                    enterprise.DBName = "BusinessDB" + DateTime.Now.Ticks;
                    context.Enterprises.Add(enterprise);
                    return context.SaveChanges() > 0;
                }
            }
            catch (Exception ex)
            {
                return false;
            }
        }

    }
}
```
XHZNL.EFDynamicDatabaseBuilding.Common.CommonHelper:
```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.Remoting.Messaging;
using System.Text;
using System.Threading.Tasks;

namespace XHZNL.EFDynamicDatabaseBuilding.Common
{
    public class CommonHelper
    {
        public static readonly CommonHelper Instance = new CommonHelper();

        private CommonHelper() { }


        /// <summary>
        /// 获取当前数据库
        /// </summary>
        /// <returns></returns>
        public string GetCurrentDBName()
        {
            var key = "CurrentDBName";

            string name = null;

            if (System.Web.HttpContext.Current != null && System.Web.HttpContext.Current.Session != null)
            {
                name = System.Web.HttpContext.Current.Session[key].ToString();
            }
            else
            {
                name = CallContext.GetData(key).ToString();
            }

            if (string.IsNullOrEmpty(name))
                throw new Exception("CurrentDBName异常");

            return name;
        }

        /// <summary>
        /// 设置当前数据库
        /// </summary>
        /// <param name="name"></param>
        public void SetCurrentDBName(string name)
        {
            var key = "CurrentDBName";

            if (System.Web.HttpContext.Current != null && System.Web.HttpContext.Current.Session != null)
            {
                System.Web.HttpContext.Current.Session[key] = name;
            }
            else
            {
                CallContext.SetData(key, name);
            }
        }
    }
}
```
web.config配置一下业务数据库的连接信息：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518152828879-846934579.png)
这个可以根据实际业务修改，分布到不同的服务器，这里只是为了演示。

业务数据库模型放在XHZNL.EFDynamicDatabaseBuilding.BusinessEntity里面，这里只有一个员工表
```
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace XHZNL.EFDynamicDatabaseBuilding.BusinessEntity
{
    /// <summary>
    /// 员工
    /// </summary>
    public class Staff
    {
        /// <summary>
        /// ID
        /// </summary>
        [Required]
        public Guid ID { get; set; }

        /// <summary>
        /// 员工名称
        /// </summary>
        [Required]
        [Column(TypeName = "NVARCHAR")]
        [MaxLength(50)]
        public string Name { get; set; }
    }
}
```
数据库context:
```
using System;
using System.Collections.Generic;
using System.Data.Entity;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace XHZNL.EFDynamicDatabaseBuilding.BusinessEntity
{

    //[DbConfigurationType(typeof(MySql.Data.Entity.MySqlEFConfiguration))]//使用mysql时需要这个
    internal class BusinessDBContext : DbContext
    {
        public BusinessDBContext() : base("name=BusinessDB")
        {
            Database.SetInitializer<BusinessDBContext>(null);
        }

        //修改上下文默认构造函数  
        public BusinessDBContext(string connectionString)
            : base(connectionString)
        {

        }

        /// <summary>
        /// 员工
        /// </summary>
        public DbSet<Staff> Staffs { get; set; }
    }
}
```
XHZNL.EFDynamicDatabaseBuilding.BusinessEntity.Migrations.Configuration:可以放一些种子数据...
```
namespace XHZNL.EFDynamicDatabaseBuilding.BusinessEntity.Migrations
{
    using System;
    using System.Data.Entity;
    using System.Data.Entity.Migrations;
    using System.Linq;

    internal sealed class Configuration : DbMigrationsConfiguration<XHZNL.EFDynamicDatabaseBuilding.BusinessEntity.BusinessDBContext>
    {
        public Configuration()
        {
            AutomaticMigrationsEnabled = true;
            AutomaticMigrationDataLossAllowed = true;
            //SetSqlGenerator("MySql.Data.MySqlClient", new MySql.Data.Entity.MySqlMigrationSqlGenerator());//使用mysql时需要这个
        }

        protected override void Seed(XHZNL.EFDynamicDatabaseBuilding.BusinessEntity.BusinessDBContext context)
        {
            //  This method will be called after migrating to the latest version.

            //  You can use the DbSet<T>.AddOrUpdate() helper extension method
            //  to avoid creating duplicate seed data.

            var staff = new Staff() { ID = Guid.Parse("212cf53c-6801-4c00-b36b-996ac9809e04"), Name = "初始员工" };
            context.Staffs.AddOrUpdate(staff);
            context.SaveChanges();
        }
    }
}
```
关键的分库，建库，更新数据库代码在XHZNL.EFDynamicDatabaseBuilding.BusinessEntity.Services.BaseService，任何的模型修改都能在程序运行时自动更新到数据库:
```
using System;
using System.Collections.Generic;
using System.Data.Entity;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using XHZNL.EFDynamicDatabaseBuilding.Common;

namespace XHZNL.EFDynamicDatabaseBuilding.BusinessEntity.Services
{
    public class BaseService
    {
        /// <summary>
        /// 获取context
        /// </summary>
        /// <returns></returns>
        internal BusinessDBContext GetDBContext()
        {
            try
            {
                //mysql连接字符串
                //var connectionString = $"Data Source={AppConfig.DB_DataSource};Port={AppConfig.DB_Port};Initial Catalog={CommonHelper.Instance.GetCurrentDBName()};User ID={AppConfig.DB_UserID};Password={AppConfig.DB_Password};";

                //sqlserver连接字符串
                var connectionString = $"Data Source={AppConfig.DB_DataSource},{AppConfig.DB_Port};Initial Catalog={CommonHelper.Instance.GetCurrentDBName()};User ID={AppConfig.DB_UserID};Password={AppConfig.DB_Password};";

                var context = new BusinessDBContext(connectionString);

                //数据库是否存在 不存在则创建
                if (!context.Database.Exists())
                {
                    context.Database.Create();

                    var dbInitializer = new MigrateDatabaseToLatestVersion<BusinessDBContext, Migrations.Configuration>(true);
                    dbInitializer.InitializeDatabase(context);
                }

                //数据库接口是否和模型一致 不一致则更新
                if (!context.Database.CompatibleWithModel(false))
                {
                    var dbInitializer = new MigrateDatabaseToLatestVersion<BusinessDBContext, Migrations.Configuration>(true);
                    dbInitializer.InitializeDatabase(context);
                }

                return context;
            }
            catch (Exception ex)
            {
                return null;
            }
        }
    }
}
```
其他的数据访问类继承BaseService，通过GetDBContext()方法获取context，这样确保得到正确的业务数据库。
# 四、效果
- 运行web项目：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518155600615-52966713.png)
此时数据库中只有一个主库：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518155739050-1853669367.png)
- 点击注册企业：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518155904937-1008332578.png)
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518160040194-794616613.png)
注册2个企业用于测试
此时主库已有了2条企业数据：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518160139211-1728642323.png)
- 分别用test1，test2登录，并添加员工数据：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518160329266-883415131.png)
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518160507400-133930254.png)
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518160657969-252626216.png)
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518160728268-773451038.png)
企业登录后已经生成了对应的业务库
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518160630140-1354819628.png)
- 数据正确添加读取：
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518161156642-1833195883.png)
![](https://img2020.cnblogs.com/blog/610959/202005/610959-20200518161245840-557293001.png)
# 五、总结：
以上关于EntityFramework分库的核心就是通过动态构建connectionString，来得到context。至于如何动态构建，方法有很多，以上代码只是最简单的实现。代码在：https://github.com/xiajingren/EFDynamicDatabaseBuilding