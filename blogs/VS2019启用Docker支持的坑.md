- 使用vs2019创建ASP.Net Core Web应用程序：
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200609222736692-1125633374.png)
- 右侧高级选项中有一项启用Docker支持，勾选后vs会自动帮我们创建Dockerfile：
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200609223303210-1438605531.png)
- 看一下Dockerfile的内容：
```
#See https://aka.ms/containerfastmode to understand how Visual Studio uses this Dockerfile to build your images for faster debugging.

FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build
WORKDIR /src
COPY ["WebApplication1/WebApplication1.csproj", "WebApplication1/"]
RUN dotnet restore "WebApplication1/WebApplication1.csproj"
COPY . .
WORKDIR "/src/WebApplication1"
RUN dotnet build "WebApplication1.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "WebApplication1.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "WebApplication1.dll"]
```
乍一看好像很完美。。。
- 下面用docker build一下，通常的做法是：
`docker build -t mywebapp .`
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200609223927843-719867775.png)
然后进行到第6步就报错了：COPY failed: stat /var/lib/docker/tmp/docker-builder893785636/WebApplication1/WebApplication1.csproj: no such file or directory
没有这样的文件或目录，仔细一看COPY ["WebApplication1/WebApplication1.csproj", "WebApplication1/"]确实不对，WebApplication1这级目录是多余的。
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200609224611105-1785928365.png)
然后我试着修改Dockerfile文件，最后怎么改都没能成功。。。

# 解决方案一：
- 把生成的Dockerfile文件移到上一级目录，在上一级目录执行`docker build -t mywebapp .`就可以了。
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200609225554001-1548511628.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200609225620268-1112975646.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200609225647604-1408145085.png)

# 解决方案二：
- 在上一级目录执行docker build并使用-f参数指定Dockerfile文件位置：`docker build -t mywebapp1 -f ./WebApplication1/Dockerfile .`
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200609230652024-1785905301.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200609230730592-1764707245.png)

# 解决方案三：
- 直接使用vs2019来启动，将项目设置为Docker启动：
![](https://img2020.cnblogs.com/blog/610959/202007/610959-20200720184413044-66730729.png)
Ctrl+F5即可启动：
![](https://img2020.cnblogs.com/blog/610959/202007/610959-20200720192221231-1442630510.png)
但是由于网络等原因，可能又会遇到其他问题。。。