# Step-by-Step Guide: Setting Up a Reverse Proxy in .NET 8 with YARP

Let’s build a reverse proxy in .NET 8 using YARP, routing requests to two external APIs (`jsonplaceholder.typicode.com` and `reqres.in`) based on a provided JSON configuration.

### Prerequisites
- .NET 8 SDK installed on your machine.
- A basic understanding of ASP.NET Core.
- An editor like Visual Studio, Visual Studio Code, or JetBrains Rider.

---

### Step 1: Create a New ASP.NET Core Project

Start by creating a new ASP.NET Core Web API project to host the reverse proxy.

```bash
dotnet new webapi -n ReverseProxyDemo
cd ReverseProxyDemo
```

This generates a basic Web API project with a `Controllers` folder, `Program.cs`, and other default files.

---

### Step 2: Install the YARP NuGet Package

YARP is available as a NuGet package. Add the `Yarp.ReverseProxy` package to your project.

```bash
dotnet add package Yarp.ReverseProxy
```

This package provides the core functionality for setting up a reverse proxy in .NET.

---

### Step 3: Configure YARP Using `appsettings.json`

We’ll use the provided JSON configuration to define routes and clusters in `appsettings.json`. Replace the default content of `appsettings.json` with the following:

```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Information",
            "Microsoft.AspNetCore": "Warning"
        }
    },
    "AllowedHosts": "*",
    "ReverseProxy": {
        "Routes": {
            "posts": {
                "ClusterId": "jsonplaceholder",
                "Match": {
                    "Path": "/posts/{**catch-all}"
                }
            },
            "reqres": {
                "ClusterId": "reqres",
                "Match": {
                    "Path": "/api/{**catch-all}"
                }
            }
        },
        "Clusters": {
            "jsonplaceholder": {
                "Destinations": {
                    "destination1": {
                        "Address": "https://jsonplaceholder.typicode.com/"
                    }
                }
            },
            "reqres": {
                "Destinations": {
                    "destination1": {
                        "Address": "https://reqres.in/"
                    }
                }
            }
        }
    }
}
```

#### Explanation of the Configuration:
- **Logging:** Configures log levels for the application (`Information` for general logs, `Warning` for ASP.NET Core logs).
- **AllowedHosts:** Allows requests from any host (`*`).
- **ReverseProxy Routes:**
  - `"posts"`: Matches requests starting with `/posts/` and routes them to the `jsonplaceholder` cluster.
  - `"reqres"`: Matches requests starting with `/api/` and routes them to the `reqres` cluster.
- **ReverseProxy Clusters:**
  - `"jsonplaceholder"`: Proxies requests to `https://jsonplaceholder.typicode.com/`.
  - `"reqres"`: Proxies requests to `https://reqres.in/`.

---

### Step 4: Configure YARP in `Program.cs`

Configure YARP in `Program.cs` to use the configuration from `appsettings.json`. Replace the default content with the following:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add YARP services and load configuration from appsettings.json
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

// Map the reverse proxy middleware
app.MapReverseProxy();

app.Run();
```

#### Explanation:
- `AddReverseProxy()`: Registers YARP services in the dependency injection container.
- `LoadFromConfig()`: Loads routes and clusters from the `ReverseProxy` section of `appsettings.json`.
- `MapReverseProxy()`: Adds the reverse proxy middleware to handle incoming requests.

---

### Step 5: Test the Reverse Proxy

Run the application.

```bash
dotnet run
```

By default, it runs on `http://localhost:5000` (or `https://localhost:5001` if HTTPS is enabled). Test the following URLs using a browser or a tool like Postman:

- `http://localhost:5000/posts/todos/1`: This proxies to `https://jsonplaceholder.typicode.com/todos/1` and should return a JSON response like:
  ```json
  {
    "userId": 1,
    "id": 1,
    "title": "delectus aut autem",
    "completed": false
  }
  ```
- `http://localhost:5000/api/users`: This proxies to `https://reqres.in/api/users` and should return a paginated list of users, like:
  ```json
  {
    "page": 1,
    "per_page": 6,
    "total": 12,
    "total_pages": 2,
    "data": [...]
  }
  ```

If these work, your reverse proxy is successfully routing requests to the backend APIs!

---

### Step 6: Add Custom Middleware (Optional)

You can inject custom logic into the request pipeline with YARP. For example, let’s add a simple logging middleware to log incoming requests. Update `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add YARP services and load configuration from appsettings.json
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

// Add custom middleware to log requests
app.Use(async (context, next) =>
{
    Console.WriteLine($"Received request: {context.Request.Path} at {DateTime.UtcNow}");
    await next(context);
});

// Map the reverse proxy middleware
app.MapReverseProxy();

app.Run();
```

This logs the path and timestamp of each incoming request before it’s proxied.

---

### Step 7: Add Load Balancing (Optional)

YARP supports load balancing across multiple destinations in a cluster. Update the `Clusters` section in `appsettings.json` to include multiple destinations and a load-balancing policy:

```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Information",
            "Microsoft.AspNetCore": "Warning"
        }
    },
    "AllowedHosts": "*",
    "ReverseProxy": {
        "Routes": {
            "posts": {
                "ClusterId": "jsonplaceholder",
                "Match": {
                    "Path": "/posts/{**catch-all}"
                }
            },
            "reqres": {
                "ClusterId": "reqres",
                "Match": {
                    "Path": "/api/{**catch-all}"
                }
            }
        },
        "Clusters": {
            "jsonplaceholder": {
                "Destinations": {
                    "destination1": {
                        "Address": "https://jsonplaceholder.typicode.com/"
                    }
                    // Add more destinations if you have multiple instances
                },
                "LoadBalancingPolicy": "RoundRobin"
            },
            "reqres": {
                "Destinations": {
                    "destination1": {
                        "Address": "https://reqres.in/"
                    }
                }
            }
        }
    }
}
```