using System.Collections.Concurrent; // for thread-safe dictionary
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using System.Web;
using System.Web.Http;
using System.Net.WebSockets;

public class WSChatController : ApiController
{
    private static readonly HashSet<WebSocket> Clients = new HashSet<WebSocket>();
    private static readonly object ClientsLock = new object();

    // Map each socket to the associated user ID
    private static readonly ConcurrentDictionary<WebSocket, string> SocketUserMap = new ConcurrentDictionary<WebSocket, string>();

    private static readonly object EditStateLock = new object();
    private static readonly Dictionary<string, string> DocumentEditors = new Dictionary<string, string>();

    public static int ConnectedUserCount
    {
        get
        {
            lock (ClientsLock)
            {
                return Clients.Count(client => client.State == WebSocketState.Open);
            }
        }
    }

    public HttpResponseMessage Get()
    {
        if (HttpContext.Current.IsWebSocketRequest)
        {
            HttpContext.Current.AcceptWebSocketRequest(ProcessWSChat);
        }
        return new HttpResponseMessage(HttpStatusCode.SwitchingProtocols);
    }

    private async Task ProcessWSChat(AspNetWebSocketContext context)
    {
        var socket = context.WebSocket;

        lock (ClientsLock)
        {
            Clients.Add(socket);
        }

        string userId = null;

        // Receive initial userId message from client
        var buffer = new ArraySegment<byte>(new byte[1024]);
        var initialResult = await socket.ReceiveAsync(buffer, CancellationToken.None);
        if (initialResult.MessageType == WebSocketMessageType.Text)
        {
            string initialMessage = Encoding.UTF8.GetString(buffer.Array, 0, initialResult.Count);
            if (initialMessage.StartsWith("userId:"))
            {
                userId = initialMessage.Substring("userId:".Length).Trim();
                SocketUserMap[socket] = userId;
            }
        }

        // If no userId sent, assign default or socket hashcode
        if (string.IsNullOrEmpty(userId))
        {
            userId = socket.GetHashCode().ToString();
            SocketUserMap[socket] = userId;
        }

        await BroadcastMessageAsync($"User {userId} connected. Users online: {ConnectedUserCount}");

        try
        {
            while (socket.State == WebSocketState.Open)
            {
                buffer = new ArraySegment<byte>(new byte[1024]);
                var result = await socket.ReceiveAsync(buffer, CancellationToken.None);

                if (result.MessageType == WebSocketMessageType.Text)
                {
                    string message = Encoding.UTF8.GetString(buffer.Array, 0, result.Count);

                    if (message.StartsWith("edit:start:"))
                    {
                        string documentId = message.Substring("edit:start:".Length);

                        lock (EditStateLock)
                        {
                            DocumentEditors[documentId] = userId;
                        }

                        await BroadcastMessageAsync($"User {userId} started editing document {documentId}.");
                    }
                    else if (message.StartsWith("edit:stop:"))
                    {
                        string documentId = message.Substring("edit:stop:".Length);

                        lock (EditStateLock)
                        {
                            if (DocumentEditors.TryGetValue(documentId, out string editor) && editor == userId)
                            {
                                DocumentEditors.Remove(documentId);
                            }
                        }

                        await BroadcastMessageAsync($"User {userId} stopped editing document {documentId}.");
                    }
                    else
                    {
                        await BroadcastMessageAsync($"User {userId}: {message}");
                    }
                }
                else if (result.MessageType == WebSocketMessageType.Close)
                {
                    break;
                }
            }
        }
        finally
        {
            lock (ClientsLock)
            {
                Clients.Remove(socket);
            }

            SocketUserMap.TryRemove(socket, out _);

            List<string> docsToRemove;
            lock (EditStateLock)
            {
                docsToRemove = DocumentEditors.Where(kv => kv.Value == userId)
                                             .Select(kv => kv.Key).ToList();
                foreach (var doc in docsToRemove)
                {
                    DocumentEditors.Remove(doc);
                }
            }

            foreach (var docId in docsToRemove)
            {
                await BroadcastMessageAsync($"User {userId} disconnected and stopped editing document {docId}.");
            }

            await BroadcastMessageAsync($"User {userId} disconnected. Users online: {ConnectedUserCount}");

            if (socket.State == WebSocketState.Open || socket.State == WebSocketState.CloseReceived)
            {
                await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", CancellationToken.None);
            }
        }
    }

    private async Task BroadcastMessageAsync(string message)
    {
        List<WebSocket> clientsSnapshot;
        byte[] responseBytes = Encoding.UTF8.GetBytes(message);

        lock (ClientsLock)
        {
            clientsSnapshot = Clients.Where(c => c.State == WebSocketState.Open).ToList();
        }

        foreach (var client in clientsSnapshot)
        {
            await client.SendAsync(new ArraySegment<byte>(responseBytes), WebSocketMessageType.Text, true, CancellationToken.None);
        }
    }
}
