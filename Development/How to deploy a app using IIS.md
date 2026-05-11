What is IIS? Internet Information Service is a flexible, secure, and manageable web server developed by Microsoft for Windows Server to host websites, services, and applications. Just a way to host dotnet application.

**How to host using IIS?**
#### 1. Install IIS
It is by default present in windows. You just have to enable it. 
How to enable: Press `window` + `R` and type `optionalfeatures`. You will be shown a new window, just tick IIS in it. Restart your PC after this.

You can verify installation by going to `http://localhost`. Would see a page kind of similar to windows 8 start menu.

#### 2. Install ASP.Net Core Hosting Bundle
This is used by IIS to run .NET apps. You can download it from Microsoft website.
Run `iisreset` in your cmd. Requires you to be admin.

Make sure that the version of your application (in `csproj`) matches the version of hosting bundle that you are downloading.

#### 3. Publish your .NET app
We can publish using:
```bash
dotnet publish -c Release -o C:\Publish\MyApp
```
Above command should be run in your project folder where your to be deployed app is present.
You can verify if the above command worked or not by running following command in `C:\Publish\MyApp`
```bash
dotnet MyApp.dll
```
You can rename `MyApp` according to your application.

#### 4. Move files to IIS folder
Go to `C:inetpub\wwwroot\` and paste the folder created in the above step. Give this folder all the required permission.
Right click -> Properties -> Security.
Edit -> Add
```text
IIS_IUSRS
```
Give all the permission to above user.

#### 5. Create website in IIS
Open `IIS Manager`  (can search for it in start). On the right hand side you would see `View Application Pools`, you can add a pool here for your application. One `Application Pool` can only run one website. On the left hand and also right hand you will see `sites`. Go there and add a new site.

While adding website, make sure to give the path of `inetpub\wwwroot` and the pool that you created just now. `5000` can be used as a port for your application. Now you can start your application from here and Browse (on right hand side) it.


#### How would any request flow in this model?
IIS does not run our app directly. It acts as a reverse proxy.

**Flow:**
1. User hits browser
2. IIS receives request 
3. IIS forwards request to Kestrel
4. Kestrel runs our .NET app
5. Response goes back

Kestrel is a lightweight, cross-platform, and high-performance open-source web server for ASP.NET Core, included by default in all .NET Core templates.

Tags:
#development 
#dotnet 
#deployment