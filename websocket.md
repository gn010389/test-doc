
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Net.WebSockets;
using System.Text;
using System.Threading.Tasks;
using System.Threading;
using System.Web;
using System.Web.Http;
using System.Web.WebSockets;
using System.Collections.Concurrent;
using System.IO;

namespace TestAPI.Controllers
{
    public class WSChatController : ApiController
    {
        private static readonly HashSet<WebSocket> Clients = new HashSet<WebSocket>();
        private static readonly object ClientsLock = new object();
        private static readonly ConcurrentDictionary<WebSocket, string> SocketUserMap = new ConcurrentDictionary<WebSocket, string>();
        private static readonly object EditStateLock = new object();
        private static readonly Dictionary<string, string> DocumentEditors = new Dictionary<string, string>();  
        private static readonly object LogLock = new object();

        private void Log(string entry)
        {
            var now = DateTime.UtcNow;
            string logFileName = string.Format("chatlog-{0:yyyyMMdd-HH}.txt", now);
            string logFilePath = HttpContext.Current.Server.MapPath("~/App_Data/" + logFileName);
            lock (LogLock)
            {
                File.AppendAllText(logFilePath, $"{now:u};{entry}{Environment.NewLine}");
            }
        }

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
            string userId = null;

            // Receive userId from client
            var buffer = new ArraySegment<byte>(new byte[1024]);
            var initialResult = await socket.ReceiveAsync(buffer, CancellationToken.None);
            if (initialResult.MessageType == WebSocketMessageType.Text)
            {
                string initialMessage = Encoding.UTF8.GetString(buffer.Array, 0, initialResult.Count);
                if (initialMessage.StartsWith("userId:"))
                {
                    userId = initialMessage.Substring("userId:".Length).Trim();
                    SocketUserMap[socket] = userId;
                    Log($"Initial connection message: userId declared as '{userId}' by socket {socket.GetHashCode()}");
                }
            }
            if (string.IsNullOrEmpty(userId))
            {
                userId = socket.GetHashCode().ToString();
                SocketUserMap[socket] = userId;
                Log($"Initial connection message: No userId sent; using default '{userId}' for socket {socket.GetHashCode()}");
            }

            lock (ClientsLock)
            {
                Clients.Add(socket);
                Log($"Connection: User {userId} connected. Total online: {ConnectedUserCount}");
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
                        Log($"Message from {userId}: {message}");

                        if (message.StartsWith("edit:start:"))
                        {
                            string documentId = message.Substring("edit:start:".Length);
                            string existingEditor = null;
                            bool canEdit = false;
                            lock (EditStateLock)
                            {
                                DocumentEditors.TryGetValue(documentId, out existingEditor);
                                if (string.IsNullOrEmpty(existingEditor))
                                {
                                    DocumentEditors[documentId] = userId;
                                    canEdit = true;
                                    Log($"Edit Start: User {userId} started editing document {documentId}");
                                }
                            }
                            if (canEdit)
                            {
                                await BroadcastMessageAsync($"User {userId} started editing document {documentId}.");
                            }
                            else if (existingEditor != userId)
                            {
                                string msg = $"Document {documentId} is already being edited by {existingEditor}";
                                Log(msg);
                                await socket.SendAsync(
                                    new ArraySegment<byte>(Encoding.UTF8.GetBytes(msg)),
                                    WebSocketMessageType.Text,
                                    true,
                                    CancellationToken.None
                                );
                            }
                        }
                        else if (message.StartsWith("edit:stop:"))
                        {
                            string documentId = message.Substring("edit:stop:".Length);
                            lock (EditStateLock)
                            {
                                if (DocumentEditors.TryGetValue(documentId, out string editor) && editor == userId)
                                {
                                    DocumentEditors.Remove(documentId);
                                    Log($"Edit Stop: User {userId} stopped editing document {documentId}");
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
                        Log($"Close frame received for user {userId} ({socket.GetHashCode()})");
                        break;
                    }
                }
            }
            finally
            {
                lock (ClientsLock)
                {
                    Clients.Remove(socket);
                    Log($"Disconnect: User {userId} disconnected. Users online: {ConnectedUserCount}");
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
                        Log($"Edit Disconnect: User {userId} disconnected and stopped editing document {doc}");
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
                    Log($"Closed websocket for user {userId} ({socket.GetHashCode()})");
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
            Log($"Broadcast: {message}");
        }
    }
}
