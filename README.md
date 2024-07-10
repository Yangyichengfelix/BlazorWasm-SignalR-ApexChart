# BlazorWasm-SignalR-ApexChart
Exemple d'utilisation d'ApexChart sur Blazor WebAssembly avec mise à jour en temps réel à l'aide de SignalR

## Introduction
SignalR est une bibliothèque gratuite et open-source pour ASP.NET Core qui permet au serveur d'envoyer des messages asynchrones en temps réel aux clients connectés. C'est une couche d'abstraction au-dessus de WebSockets, facilitant son utilisation et fournissant des alternatives à d'autres formes de communication lorsque cela est nécessaire (événements envoyés par le serveur et sondage long).

Ce projet montre comment créer une application Blazor WebAssembly qui affiche des graphiques en temps réel à partir d'un serveur SignalR.


## Structure du projet

### Le projet comprendra 4 sous-projets, créés à partir de 2 modèles de projet :

  - Projet ASP.NET Core (pour le serveur SignalR)
    - BlazorWasmSignalR.SignalRServer
  - Application Blazor WebAssembly (hébergée par ASP.NET Core)
    - BlazorWasmSignalR.Wasm.Client (Blazor WASM)
    - BlazorWasmSignalR.Wasm.Server (hôte ASP.NET Core)
    - BlazorWasmSignalR.Wasm.Shared (composants communs)

## Le Backend - Serveur SignalR

SignalR fait partie d'ASP.NET Core. Pour l'utiliser, il suffit de le configurer dans notre fichier Startup.cs ou Program.cs (si l'on utilise des instructions de haut niveau). Les clients SignalR se connectent à des Hubs, qui sont des composants définissant des méthodes pouvant être appelées par les clients pour envoyer des messages ou s'abonner à des messages provenant du serveur.

``` csharp
/*
* BlazorWasmSignalR.SignalRServer
* Program.cs
*/
//Add SignalR services
builder.Services.AddSignalR();

...

var app = builder.Build();

...

//Map our Hub
app.MapHub<RealTimeDataHub>("/realtimedata");

app.Run();

```
Dans cette démo, on créé un Hub avec deux méthodes qui retournent chacune un flux de données simulant une variation du prix d'une devise. Lorsque le client appelle ces méthodes, son ConnectionId est ajouté à une liste d'auditeurs et le flux de données est retourné au client. Ensuite, le service RealTimeDataStreamWriter écrit chaque changement de prix de la devise dans les flux des auditeurs.

La méthode OnDisconnectedAsync est appelée lorsqu'un client se déconnecte, retirant ainsi le client de la liste des auditeurs.

⚠️ Il s'agit d'une implémentation basique qui n'est pas évolutive horizontalement. Elle est uniquement à des fins de démonstration. Pour une mise à l'échelle horizontale de SignalR, un [dataplane Redis](https://learn.microsoft.com/en-us/aspnet/core/signalr/redis-backplane?view=aspnetcore-8.0) doit être configuré.

[Documentation Redis Backplane SignalR](https://github.com/dotnet/AspNetCore.Docs/blob/main/aspnetcore/signalr/redis-backplane.md)

### RealTimeDataHub implémentation
```csharp
/*
* BlazorWasmSignalR.SignalRServer.Hubs
* RealTimeDataHub.cs
*/
using BlazorWasmSignalR.SignalRServer.BackgroundServices;
using BlazorWasmSignalR.Wasm.Shared;
using Microsoft.AspNetCore.SignalR;
using System.Threading.Channels;

namespace BlazorWasmSignalR.SignalRServer.Hubs;

public class RealTimeDataHub : Hub
{
    private readonly RealTimeDataStreamWriter _realTimeDataStreamWriter;

    public RealTimeDataHub(RealTimeDataStreamWriter realTimeDataStreamWriter)
    {
        _realTimeDataStreamWriter = realTimeDataStreamWriter;
    }

    public ChannelReader<CurrencyStreamItem> CurrencyValues(CancellationToken cancellationToken)
    {
        var channel = Channel.CreateUnbounded<CurrencyStreamItem>();

        _realTimeDataStreamWriter.AddCurrencyListener(Context.ConnectionId, channel.Writer);

        return channel.Reader;
    }

    public ChannelReader<DataItem> Variation(CancellationToken cancellationToken)
    {
        var channel = Channel.CreateUnbounded<DataItem>();

        _realTimeDataStreamWriter.AddVariationListener(Context.ConnectionId, channel.Writer);

        return channel.Reader;
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        _realTimeDataStreamWriter.RemoveListeners(Context.ConnectionId);

        await base.OnDisconnectedAsync(exception);
    }
}

```

### RealTimeDataStreamWriter implémentation 
Ce service maintient une liste de clients abonnés pour recevoir les changements de prix et simule un changement de prix chaque seconde.
```csharp
/*
* BlazorWasmSignalR.SignalRServer.BackgroundServices
* RealTimeDataStreamWriter.cs
*/
using BlazorWasmSignalR.Wasm.Shared;
using System.Security.Cryptography;
using System.Threading.Channels;

namespace BlazorWasmSignalR.SignalRServer.BackgroundServices;

public class RealTimeDataStreamWriter
{
    private readonly Dictionary<string, ChannelWriter<CurrencyStreamItem>> _currencyWriters;
    private readonly Dictionary<string, ChannelWriter<DataItem>> _variationWriters;
    private readonly Timer _timer = default!;

    private int _currentVariationValue = 50;
    private decimal _currentYenValue = RandomNumberGenerator.GetInt32(1, 3);
    private decimal _currentEuroValue = RandomNumberGenerator.GetInt32(1, 3);

    public RealTimeDataStreamWriter()
    {
        _timer = new(OnElapsedTime, null, TimeSpan.Zero, TimeSpan.FromSeconds(1));

        _currencyWriters = new();
        _variationWriters = new();
    }

    public void AddCurrencyListener(string connectionId, ChannelWriter<CurrencyStreamItem> channelWriter)
    {
        _currencyWriters[connectionId] = channelWriter;
    }

    public void AddVariationListener(string connectionId, ChannelWriter<DataItem> channelWriter)
    {
        _variationWriters[connectionId] = channelWriter;
    }

    public void RemoveListeners(string connectionId)
    {
        _currencyWriters.Remove(connectionId);
        _variationWriters.Remove(connectionId);
    }

    public void Dispose()
    {
        _timer?.Dispose();
    }

    private void OnElapsedTime(object? state)
    {
        SendCurrencyData();
        SendVariationData();
    }

    private void SendCurrencyData()
    {
        var date = DateTime.Now;

        var yenDecimals = RandomNumberGenerator.GetInt32(-20, 20) / 100M;
        var euroDecimals = RandomNumberGenerator.GetInt32(-20, 20) / 100M;

        _currentYenValue = Math.Max(0.5M, _currentYenValue + yenDecimals);
        _currentEuroValue = Math.Max(0.5M, _currentEuroValue + euroDecimals);

        var currencyStreamItem = new CurrencyStreamItem()
        {
            Minute = date.ToString("hh:mm:ss"),
            YenValue = _currentYenValue,
            EuroValue = _currentEuroValue
        };

        foreach(var listener in _currencyWriters)
        {
            _ = listener.Value.WriteAsync(currencyStreamItem);
        }
    }

    private void SendVariationData()
    {
        var min = Math.Max(0, _currentVariationValue - 20);
        var max = Math.Min(100, _currentVariationValue + 20);

        var variationValue = new DataItem(DateTime.Now.ToString("hh:mm:ss"),
            RandomNumberGenerator.GetInt32(min, max));

        _currentVariationValue = (int)variationValue.Value;

        foreach (var listener in _variationWriters)
        {
            _ = listener.Value.WriteAsync(variationValue);
        }
    }
}

```
### ApexCharts pour Blazor

ApexCharts est une bibliothèque JavaScript gratuite et open-source permettant de générer des graphiques interactifs et réactifs. Elle propose une large gamme de types de graphiques et est particulièrement adaptée pour les graphiques en temps réel grâce à ses animations fluides.

ApexCharts pour Blazor est une bibliothèque wrapper permettant d'utiliser ApexCharts dans les applications Blazor. Elle fournit un ensemble de composants Blazor facilitant l'intégration des graphiques ApexCharts au sein des applications Blazor.

## Frontend - Blazor WebAssembly
Dans le projet Blazor WebAssembly, nous devons installer les packages NuGet Blazor-ApexCharts et Microsoft.AspNetCore.SignalR.Client :

```bash
dotnet add package Blazor-ApexCharts
dotnet add package Microsoft.AspNetCore.SignalR.Client
```

### Ajouter les graphiques
Sur la page principale, nous aurons deux graphiques :
  - Un graphique en ligne avec des mises à jour de prix pour le Yen et l’Euro ;
  - Un graphique de jauge avec une valeur de variation.
Pour afficher un graphique à l’aide d’ApexCharts pour Blazor, nous utilisons le composant ApexChart et un ApexPointSeries pour chaque série.

```razor
@* BlazorWasmSignalR.Wasm.Client.Pages*@
@* RealtimeCharts.razor *@
<ApexChart TItem="DataItem"
           Title="Currency Exchange Rates in USD"
           Options="@_lineChartOptions"
           @ref="_lineChart">

    <ApexPointSeries TItem="DataItem"
                     Items="_yenSeries"
                     Name="Yen"
                     SeriesType="SeriesType.Line"
                     XValue="@(e => e.Minute)"
                     YAggregate="@(e => e.Sum(e => e.Value))" />

    <ApexPointSeries TItem="DataItem"
                     Items="_euroSeries"
                     Name="Euro"
                     SeriesType="SeriesType.Line"
                     XValue="@(e => e.Minute)"
                     YAggregate="@(e => e.Sum(e => e.Value))" />
</ApexChart>

```
ℹ️ L'attribut @ref définit une variable pour accéder à l'objet ApexChart. Il sera utilisé pour mettre à jour le graphique lorsque de nouvelles valeurs arrivent.

La propriété Options reçoit un ApexChartOptions<DataItem> où nous pouvons personnaliser notre graphique. Dans cet exemple, on :

  - Active les animations et règle leur vitesse à 1 seconde ;
  - Désactive la barre d'outils et le zoom du graphique ;
  - Verrouille l'axe X à 12 éléments. Les anciennes valeurs sont supprimées du graphique ;
  - Fixe l'axe Y dans la plage de 0 à 5.

```csharp
@* BlazorWasmSignalR.Wasm.Client.Pages*@
@* RealtimeCharts.razor.cs *@
private ApexChartOptions<DataItem> _lineChartOptions = new ApexChartOptions<DataItem>
{
    Chart = new Chart
    {
        Animations = new()
        {
            Enabled = true,
            Easing = Easing.Linear,
            DynamicAnimation = new()
            {
                Speed = 1000
            }
        },
        Toolbar = new()
        {
            Show = false
        },
        Zoom = new()
        { 
            Enabled = false
        }
    },
    Stroke = new Stroke { Curve = Curve.Straight },
    Xaxis = new()
    {
        Range = 12
    },
    Yaxis = new()
    {
        new()
        {
            DecimalsInFloat = 2,
            TickAmount = 5,
            Min = 0,
            Max = 5
        }
    }
};
```


💡 La documentation des options se trouve dans [les documents d'ApexCharts](https://apexcharts.com/docs/options/annotations/).

### Implémentation de RealtimeCharts.razor

```razor
@page "/"
@using BlazorWasmSignalR.Wasm.Shared
@using System.Security.Cryptography;
@* BlazorWasmSignalR.Wasm.Client.Pages*@
@* RealtimeCharts.razor *@
<PageTitle>Real-time charts in Blazor WebAssembly</PageTitle>

<h1>Real-time charts in Blazor WebAssembly</h1>

<div class="chart-container">
    <div class="radial-chart">
        <ApexChart TItem="DataItem"
                   Title="Transactions"
                   Options="@_radialChartOptions"
                   @ref="_radialChart">

            <ApexPointSeries TItem="DataItem"
                             Items="_radialData"
                             SeriesType="SeriesType.RadialBar"
                             Name="Variation"
                             XValue="@(e => "Variation")"
                             YAggregate="@(e => e.Average(e => e.Value))" />
        </ApexChart>
    </div>

    <div class="line-chart">
        <ApexChart TItem="DataItem"
                   Title="Currency Exchange Rates in USD"
                   Options="@_lineChartOptions"
                   @ref="_lineChart">

            <ApexPointSeries TItem="DataItem"
                             Items="_yenSeries"
                             Name="Yen"
                             SeriesType="SeriesType.Line"
                             XValue="@(e => e.Minute)"
                             YAggregate="@(e => e.Sum(e => e.Value))" />

            <ApexPointSeries TItem="DataItem"
                             Items="_euroSeries"
                             Name="Euro"
                             SeriesType="SeriesType.Line"
                             XValue="@(e => e.Minute)"
                             YAggregate="@(e => e.Sum(e => e.Value))" />
        </ApexChart>
    </div>
</div>

```

### Connexion au stream SignalR
Pour se connecter à un flux SignalR, nous créons une connexion au Hub en utilisant la classe HubConnectionBuilder et ouvrons la connexion à l'aide de la méthode StartAsync de la classe HubConnection.

Ensuite, nous nous abonnons au flux en utilisant la méthode StreamAsChannelAsync, en passant le nom du flux.

```csharp
@* BlazorWasmSignalR.Wasm.Client.Pages*@
@* RealtimeCharts.razor.cs *@
var connection = new HubConnectionBuilder()
    .WithUrl(_configuration["RealtimeDataUrl"]!) //https://localhost:7086/realtimedata
    .Build();

await connection.StartAsync();

var channelCurrencyStreamItem = await connection
    .StreamAsChannelAsync<CurrencyStreamItem>("CurrencyValues");

var channelVariation = await connection
    .StreamAsChannelAsync<DataItem>("Variation");

```
ℹ️ Notez qu'une seule connexion peut être utilisée pour s'abonner à plusieurs flux.

### Mise à jour des valeurs du graphique en temps réel
Pour lire les données du flux, on utilise la méthode WaitToReadAsync de la classe ChannelReader pour attendre de nouveaux messages, puis on les parcourt avec la méthode TryRead.

Ensuite, on ajoute les valeurs à la série et on appelle la méthode UpdateSeriesAsync du graphique pour forcer un nouveau rendu.

```csharp
@* BlazorWasmSignalR.Wasm.Client.Pages*@
@* RealtimeCharts.razor.cs *@
private async Task ReadCurrencyStreamAsync(ChannelReader<CurrencyStreamItem> channelCurrencyStreamItem)
{
    // Wait asynchronously for data to become available
    while (await channelCurrencyStreamItem.WaitToReadAsync())
    {
        // Read all currently available data synchronously, before waiting for more data
        while (channelCurrencyStreamItem.TryRead(out var currencyStreamItem))
        {
            _yenSeries.Add(new(currencyStreamItem.Minute, currencyStreamItem.YenValue));
            _euroSeries.Add(new(currencyStreamItem.Minute, currencyStreamItem.EuroValue));

            await _lineChart.UpdateSeriesAsync();
        }
    }
}

```


todo:


### Auteur d'origine
[Projet d'origine: Daniel Genezini](https://blog.genezini.com/p/real-time-charts-with-blazor-signalr-and-apexcharts/)
