Now we have created a web api using dotnet [Creating Web Api in dotnet](Creating%20Web%20Api%20in%20dotnet.md) but we need to consume this api in some frontend. The most basic version which you can do in dotnet is Razor pages. By doing using it this way, you don't even need to setup cors on your api because razor pages is making request from the server not the browser `HttpClient` in `program.cs`.

Steps to be followed:
#### 1. Create project
```bash
dotnet new webapp -n Client

cd Client
```
You can open in vs code or visual studio, your choice.

#### 2. Configure HttpClient (connect to your web api)
Update the `program.cs` that comes by default:
```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
// Using razor pages.
builder.Services.AddRazorPages();  

builder.Services.AddHttpClient("MyWebApi", client =>
{
    client.BaseAddress = new Uri("http://localhost:5241/api/");
});  

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    
    app.UseHsts();
} 

app.UseHttpsRedirection();

app.UseRouting();

app.UseAuthorization(); 

app.MapStaticAssets();

app.MapRazorPages()
   .WithStaticAssets(); 

app.Run();
```
In the line `builder.Services.AddHttpClient` we have used `MyWebApi`, this should match the project name which is running your web api.

#### 3. Create pages
Now you need to create some frontend, where you can display that data. You will have a `Pages` folder by default add below files inside it.
```cshtml
<!-- 'Pages/Prodcut.cshtml' -->
@page
@model ProductClient.Pages.ProductsModel  

<h2>Products</h2> 

<ul>
@foreach (var product in Model.Products)
{
    <li>@product.Name</li>
}
</ul>
```

```cs
// 'Pages/Product.cshtml.cs'
using Microsoft.AspNetCore.Mvc.RazorPages;
using System.Text.Json;  

namespace ProductClient.Pages
{
    public class ProductsModel : PageModel
    {
        private readonly IHttpClientFactory _clientFactory;  

        public ProductsModel(IHttpClientFactory clientFactory)
        {
            _clientFactory = clientFactory;
        } 

        public List<Product> Products { get; set; } = new();  

        public async Task OnGetAsync()
        {
			// Your project name which is hosting your web api
            var client = _clientFactory.CreateClient("MyWebApi");
			
			// What we are getting from the web api
            var response = await client.GetAsync("products"); 

            if (response.IsSuccessStatusCode)
            {
                var json = await response.Content.ReadAsStringAsync();  

                Products = JsonSerializer.Deserialize<List<Product>>(json,
                    new JsonSerializerOptions
                    {
                        PropertyNameCaseInsensitive = true
                    }) ?? new();
            }
        }
    }  

    public class Product
    {
        public int Id { get; set; }
        public string Name { get; set; } = string.Empty;
    }
}
```

In the above pages we are only displaying the data from the api, nothing fancy.

#### 4. Run your application
Now, you can go ahead and run your application using `dotnet run` and go to the specified port and view result.
`localhost:<port>/Products`


#### Short Notes
1. Create project
2. Configure `HttpClient` ( connect with your api)
3. Create pages
4. Run your application

Tags:
#development 
#dotnet 