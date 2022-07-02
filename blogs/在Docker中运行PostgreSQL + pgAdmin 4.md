- 拉取postgresql镜像：`docker pull postgres`

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617203920970-915246414.png)

- 运行postgresql：`docker run -d -p 5432:5432 --name postgresql -v pgdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=pg123456 postgres`

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617204400596-794045428.png)

- 拉取postgresql可视化工具pgadmin4：`docker pull dpage/pgadmin4`

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617204810101-1728802846.png)

- 运行pgadmin4：`docker run -d -p 5433:80 --name pgadmin4 -e PGADMIN_DEFAULT_EMAIL=test@123.com -e PGADMIN_DEFAULT_PASSWORD=123456 dpage/pgadmin4`

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617205433938-120490794.png)

- 打开浏览器访问pgadmin4：http://localhost:5433/

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617205545052-434357931.png)

- 输入我们设置的邮箱test@123.com和密码123456，点击Login

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617212205421-478746671.png)

- 连接server：

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617212428419-45985894.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617212704599-547040.png)
![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617212754797-1144917896.png)
默认username是postgres，password是上面设置的pg123456
注意，因为pgadmin运行在docker里，所以host不能写localhost。host.docker.internal代表宿主机器，或者用宿主机IP。

![](https://img2020.cnblogs.com/blog/610959/202006/610959-20200617212824906-514888659.png)
连接成功，完成！