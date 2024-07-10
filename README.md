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
[Projet d'origine: Daniel Genezini](https://blog.genezini.com/p/real-time-charts-with-blazor-signalr-and-apexcharts/)
