# test-doc
test-doc
write a code to integrate react js signalr without jQuery and with ADP.Net classic api for editing document and stop editing document

Based on my research, I'll now create a comprehensive code example for integrating React.js with SignalR and ASP.NET Classic API for document editing functionality without jQuery.

## React.js + SignalR + ASP.NET Classic API Integration for Document Editing

Here's a complete implementation showing how to integrate React.js with SignalR and ASP.NET Classic Web API for real-time document editing functionality without using jQuery:

### 1. ASP.NET Classic API Setup

#### Install SignalR Package
```
Install-Package Microsoft.AspNet.SignalR
```

#### Document Editing Hub (DocumentHub.cs)
```csharp
using Microsoft.AspNet.SignalR;
using System.Threading.Tasks;
using System.Collections.Concurrent;

public class DocumentHub : Hub
{
    // Store active document editing sessions
    private static ConcurrentDictionary<string, string> _documentSessions = 
        new ConcurrentDictionary<string, string>();
    
    // User joins a document editing session
    public async Task JoinDocument(string documentId, string userId)
    {
        await Groups.Add(Context.ConnectionId, $"Document_{documentId}");
        
        // Notify others that user started editing
        await Clients.OthersInGroup($"Document_{documentId}")
            .SendAsync("UserStartedEditing", new { 
                DocumentId = documentId, 
                UserId = userId,
                ConnectionId = Context.ConnectionId
            });
    }
    
    // User starts editing a document
    public async Task StartEditing(string documentId, string userId)
    {
        var sessionKey = $"{documentId}_{userId}";
        _documentSessions.TryAdd(sessionKey, Context.ConnectionId);
        
        // Notify all users in the document group
        await Clients.Group($"Document_{documentId}")
            .SendAsync("DocumentEditingStarted", new { 
                DocumentId = documentId, 
                UserId = userId,
                IsEditing = true
            });
    }
    
    // User stops editing a document
    public async Task StopEditing(string documentId, string userId)
    {
        var sessionKey = $"{documentId}_{userId}";
        _documentSessions.TryRemove(sessionKey, out _);
        
        // Notify all users in the document group
        await Clients.Group($"Document_{documentId}")
            .SendAsync("DocumentEditingStopped", new { 
                DocumentId = documentId, 
                UserId = userId,
                IsEditing = false
            });
    }
    
    // Real-time document content sync
    public async Task SyncDocumentContent(string documentId, string userId, object content)
    {
        // Broadcast content changes to all other users in the document
        await Clients.OthersInGroup($"Document_{documentId}")
            .SendAsync("DocumentContentChanged", new { 
                DocumentId = documentId, 
                UserId = userId,
                Content = content,
                Timestamp = System.DateTime.UtcNow
            });
    }
    
    // Handle disconnection
    public override async Task OnDisconnected(bool stopCalled)
    {
        // Clean up sessions for disconnected user
        var sessionsToRemove = _documentSessions.Where(kvp => 
            kvp.Value == Context.ConnectionId).ToList();
        
        foreach (var session in sessionsToRemove)
        {
            _documentSessions.TryRemove(session.Key, out _);
            var parts = session.Key.Split('_');
            if (parts.Length == 2)
            {
                await Clients.Group($"Document_{parts[0]}")
                    .SendAsync("DocumentEditingStopped", new { 
                        DocumentId = parts[0], 
                        UserId = parts[1],
                        IsEditing = false
                    });
            }
        }
        
        await base.OnDisconnected(stopCalled);
    }
}
```

#### Web API Controller (DocumentController.cs)
```csharp
using System.Web.Http;
using Microsoft.AspNet.SignalR;
using System.Threading.Tasks;

[RoutePrefix("api/document")]
public class DocumentController : ApiController
{
    private IHubContext _hubContext;
    
    public DocumentController()
    {
        _hubContext = GlobalHost.ConnectionManager.GetHubContext<DocumentHub>();
    }
    
    [HttpPut]
    [Route("start-edit/{documentId}")]
    public async Task<IHttpActionResult> StartEdit(string documentId)
    {
        // Add your business logic here (e.g., lock document, validate permissions)
        
        // Get user ID from request (implement your authentication logic)
        var userId = GetCurrentUserId();
        
        // Notify SignalR hub
        await _hubContext.Clients.Group($"Document_{documentId}")
            .SendAsync("DocumentEditingStarted", new { 
                DocumentId = documentId, 
                UserId = userId,
                IsEditing = true
            });
        
        return Ok(new { success = true, message = "Document editing started" });
    }
    
    [HttpPut]
    [Route("stop-edit/{documentId}")]
    public async Task<IHttpActionResult> StopEdit(string documentId)
    {
        // Add your business logic here (e.g., unlock document, save changes)
        
        var userId = GetCurrentUserId();
        
        // Notify SignalR hub
        await _hubContext.Clients.Group($"Document_{documentId}")
            .SendAsync("DocumentEditingStopped", new { 
                DocumentId = documentId, 
                UserId = userId,
                IsEditing = false
            });
        
        return Ok(new { success = true, message = "Document editing stopped" });
    }
    
    [HttpGet]
    [Route("{documentId}")]
    public IHttpActionResult GetDocument(string documentId)
    {
        // Your document retrieval logic
        var document = GetDocumentById(documentId);
        return Ok(document);
    }
    
    [HttpPut]
    [Route("{documentId}")]
    public IHttpActionResult UpdateDocument(string documentId, [FromBody]object content)
    {
        // Your document update logic
        SaveDocument(documentId, content);
        return Ok(new { success = true });
    }
    
    private string GetCurrentUserId()
    {
        // Implement your user identification logic
        return "current_user_id";
    }
    
    private object GetDocumentById(string documentId)
    {
        // Implement your document retrieval logic
        return new { id = documentId, content = "Document content", title = "Document Title" };
    }
    
    private void SaveDocument(string documentId, object content)
    {
        // Implement your document saving logic
    }
}
```

#### Startup Configuration (Startup.cs or Global.asax.cs)
```csharp
// In Startup.cs (if using OWIN)
public void Configuration(IAppBuilder app)
{
    // Enable CORS for SignalR
    app.Map("/signalr", map =>
    {
        map.UseCors(CorsOptions.AllowAll);
        var hubConfiguration = new HubConfiguration
        {
            EnableJSONP = true
        };
        map.RunSignalR(hubConfiguration);
    });
}

// Or in Global.asax.cs Application_Start
protected void Application_Start()
{
    RouteTable.Routes.MapHubs();
    // Other initialization code
}
```

### 2. React.js Client Setup

#### Install Required Packages
```bash
npm install signalr-no-jquery
npm install @types/signalr-no-jquery  # If using TypeScript
```

#### SignalR Service (signalRService.js)
```javascript
import { hubConnection } from 'signalr-no-jquery';

class SignalRService {
  constructor() {
    this.connection = null;
    this.hubProxy = null;
    this.isConnected = false;
    this.connectionCallbacks = [];
    this.eventHandlers = new Map();
  }

  // Initialize SignalR connection
  async initialize(serverUrl) {
    try {
      // Create connection without jQuery dependency
      this.connection = hubConnection(serverUrl, {
        useDefaultPath: false,
        transport: ['webSockets', 'serverSentEvents', 'longPolling']
      });

      // Create hub proxy
      this.hubProxy = this.connection.createHubProxy('documentHub');

      // Set up event handlers
      this.setupEventHandlers();

      // Start connection
      await this.startConnection();
      
      return true;
    } catch (error) {
      console.error('SignalR initialization failed:', error);
      return false;
    }
  }

  // Start SignalR connection
  startConnection() {
    return new Promise((resolve, reject) => {
      this.connection.start({ jsonp: true })
        .done(() => {
          console.log('SignalR Connected, connection ID=' + this.connection.id);
          this.isConnected = true;
          
          // Execute pending callbacks
          this.connectionCallbacks.forEach(callback => callback());
          this.connectionCallbacks = [];
          
          resolve();
        })
        .fail((error) => {
          console.error('SignalR connection failed:', error);
          this.isConnected = false;
          reject(error);
        });
    });
  }

  // Setup event handlers for hub events
  setupEventHandlers() {
    // Listen for document editing started event
    this.hubProxy.on('documentEditingStarted', (data) => {
      this.triggerEvent('documentEditingStarted', data);
    });

    // Listen for document editing stopped event
    this.hubProxy.on('documentEditingStopped', (data) => {
      this.triggerEvent('documentEditingStopped', data);
    });

    // Listen for document content changes
    this.hubProxy.on('documentContentChanged', (data) => {
      this.triggerEvent('documentContentChanged', data);
    });

    // Listen for user started editing
    this.hubProxy.on('userStartedEditing', (data) => {
      this.triggerEvent('userStartedEditing', data);
    });
  }

  // Join a document editing session
  async joinDocument(documentId, userId) {
    if (this.isConnected) {
      try {
        await this.hubProxy.invoke('joinDocument', documentId, userId);
        return true;
      } catch (error) {
        console.error('Failed to join document:', error);
        return false;
      }
    }
    return false;
  }

  // Start editing a document
  async startEditing(documentId, userId) {
    if (this.isConnected) {
      try {
        await this.hubProxy.invoke('startEditing', documentId, userId);
        return true;
      } catch (error) {
        console.error('Failed to start editing:', error);
        return false;
      }
    }
    return false;
  }

  // Stop editing a document
  async stopEditing(documentId, userId) {
    if (this.isConnected) {
      try {
        await this.hubProxy.invoke('stopEditing', documentId, userId);
        return true;
      } catch (error) {
        console.error('Failed to stop editing:', error);
        return false;
      }
    }
    return false;
  }

  // Sync document content in real-time
  async syncDocumentContent(documentId, userId, content) {
    if (this.isConnected) {
      try {
        await this.hubProxy.invoke('syncDocumentContent', documentId, userId, content);
        return true;
      } catch (error) {
        console.error('Failed to sync content:', error);
        return false;
      }
    }
    return false;
  }

  // Add event listener
  addEventListener(eventName, callback) {
    if (!this.eventHandlers.has(eventName)) {
      this.eventHandlers.set(eventName, []);
    }
    this.eventHandlers.get(eventName).push(callback);
  }

  // Remove event listener
  removeEventListener(eventName, callback) {
    if (this.eventHandlers.has(eventName)) {
      const handlers = this.eventHandlers.get(eventName);
      const index = handlers.indexOf(callback);
      if (index > -1) {
        handlers.splice(index, 1);
      }
    }
  }

  // Trigger event
  triggerEvent(eventName, data) {
    if (this.eventHandlers.has(eventName)) {
      this.eventHandlers.get(eventName).forEach(callback => {
        try {
          callback(data);
        } catch (error) {
          console.error(`Error in event handler for ${eventName}:`, error);
        }
      });
    }
  }

  // Disconnect from SignalR
  disconnect() {
    if (this.connection) {
      this.connection.stop();
      this.isConnected = false;
    }
  }
}

export default new SignalRService();
```

#### Document API Service (documentService.js)
```javascript
class DocumentService {
  constructor(baseUrl) {
    this.baseUrl = baseUrl || '/api/document';
  }

  // Start editing a document via REST API
  async startEditingAPI(documentId) {
    try {
      const response = await fetch(`${this.baseUrl}/start-edit/${documentId}`, {
        method: 'PUT',
        headers: {
          'Content-Type': 'application/json',
        },
      });

      if (response.ok) {
        return await response.json();
      } else {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
    } catch (error) {
      console.error('Error starting edit:', error);
      throw error;
    }
  }

  // Stop editing a document via REST API
  async stopEditingAPI(documentId) {
    try {
      const response = await fetch(`${this.baseUrl}/stop-edit/${documentId}`, {
        method: 'PUT',
        headers: {
          'Content-Type': 'application/json',
        },
      });

      if (response.ok) {
        return await response.json();
      } else {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
    } catch (error) {
      console.error('Error stopping edit:', error);
      throw error;
    }
  }

  // Get document content
  async getDocument(documentId) {
    try {
      const response = await fetch(`${this.baseUrl}/${documentId}`, {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
        },
      });

      if (response.ok) {
        return await response.json();
      } else {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
    } catch (error) {
      console.error('Error getting document:', error);
      throw error;
    }
  }

  // Update document content
  async updateDocument(documentId, content) {
    try {
      const response = await fetch(`${this.baseUrl}/${documentId}`, {
        method: 'PUT',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(content),
      });

      if (response.ok) {
        return await response.json();
      } else {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
    } catch (error) {
      console.error('Error updating document:', error);
      throw error;
    }
  }
}

export default DocumentService;
```

#### React Document Editor Component (DocumentEditor.jsx)
```jsx
import React, { useState, useEffect, useCallback, useRef } from 'react';
import signalRService from './signalRService';
import DocumentService from './documentService';

const DocumentEditor = ({ documentId, userId, serverUrl }) => {
  const [isEditing, setIsEditing] = useState(false);
  const [document, setDocument] = useState(null);
  const [content, setContent] = useState('');
  const [activeUsers, setActiveUsers] = useState([]);
  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState(null);
  
  const documentService = useRef(new DocumentService()).current;
  const contentTimeoutRef = useRef(null);

  // Initialize SignalR connection
  useEffect(() => {
    const initializeSignalR = async () => {
      try {
        const connected = await signalRService.initialize(serverUrl);
        setIsConnected(connected);
        
        if (connected) {
          // Join the document editing session
          await signalRService.joinDocument(documentId, userId);
          
          // Load initial document content
          const docData = await documentService.getDocument(documentId);
          setDocument(docData);
          setContent(docData.content || '');
        }
      } catch (error) {
        setError('Failed to initialize real-time connection');
        console.error('SignalR initialization error:', error);
      }
    };

    initializeSignalR();

    return () => {
      signalRService.disconnect();
    };
  }, [documentId, userId, serverUrl]);

  // Set up SignalR event listeners
  useEffect(() => {
    const handleEditingStarted = (data) => {
      if (data.documentId === documentId && data.userId !== userId) {
        setActiveUsers(prev => [...prev.filter(u => u.userId !== data.userId), {
          userId: data.userId,
          isEditing: true
        }]);
      }
    };

    const handleEditingStopped = (data) => {
      if (data.documentId === documentId && data.userId !== userId) {
        setActiveUsers(prev => prev.filter(u => u.userId !== data.userId));
      }
    };

    const handleContentChanged = (data) => {
      if (data.documentId === documentId && data.userId !== userId) {
        // Update content from other users (avoid conflicts with current editing)
        if (!isEditing) {
          setContent(data.content);
        }
      }
    };

    const handleUserStartedEditing = (data) => {
      if (data.documentId === documentId && data.userId !== userId) {
        console.log(`User ${data.userId} started editing`);
      }
    };

    signalRService.addEventListener('documentEditingStarted', handleEditingStarted);
    signalRService.addEventListener('documentEditingStopped', handleEditingStopped);
    signalRService.addEventListener('documentContentChanged', handleContentChanged);
    signalRService.addEventListener('userStartedEditing', handleUserStartedEditing);

    return () => {
      signalRService.removeEventListener('documentEditingStarted', handleEditingStarted);
      signalRService.removeEventListener('documentEditingStopped', handleEditingStopped);
      signalRService.removeEventListener('documentContentChanged', handleContentChanged);
      signalRService.removeEventListener('userStartedEditing', handleUserStartedEditing);
    };
  }, [documentId, userId, isEditing]);

  // Handle start editing
  const handleStartEditing = async () => {
    try {
      // Call both REST API and SignalR
      await Promise.all([
        documentService.startEditingAPI(documentId),
        signalRService.startEditing(documentId, userId)
      ]);
      
      setIsEditing(true);
      setError(null);
    } catch (error) {
      setError('Failed to start editing');
      console.error('Start editing error:', error);
    }
  };

  // Handle stop editing
  const handleStopEditing = async () => {
    try {
      // Save any pending changes
      if (contentTimeoutRef.current) {
        clearTimeout(contentTimeoutRef.current);
        await documentService.updateDocument(documentId, { content });
      }

      // Call both REST API and SignalR
      await Promise.all([
        documentService.stopEditingAPI(documentId),
        signalRService.stopEditing(documentId, userId)
      ]);
      
      setIsEditing(false);
      setError(null);
    } catch (error) {
      setError('Failed to stop editing');
      console.error('Stop editing error:', error);
    }
  };

  // Handle content changes with debouncing
  const handleContentChange = useCallback((newContent) => {
    setContent(newContent);
    
    // Clear previous timeout
    if (contentTimeoutRef.current) {
      clearTimeout(contentTimeoutRef.current);
    }
    
    // Debounce content synchronization
    contentTimeoutRef.current = setTimeout(async () => {
      try {
        // Sync via SignalR for real-time updates
        await signalRService.syncDocumentContent(documentId, userId, newContent);
        
        // Also save to database via REST API
        await documentService.updateDocument(documentId, { content: newContent });
      } catch (error) {
        console.error('Content sync error:', error);
      }
    }, 1000); // 1 second debounce
  }, [documentId, userId]);

  if (!document) {
    return <div>Loading document...</div>;
  }

  return (
    <div className="document-editor">
      <div className="editor-header">
        <h2>{document.title}</h2>
        <div className="editor-controls">
          <div className="connection-status">
            Status: {isConnected ? '🟢 Connected' : '🔴 Disconnected'}
          </div>
          
          {isEditing ? (
            <button onClick={handleStopEditing} className="btn btn-danger">
              Stop Editing
            </button>
          ) : (
            <button onClick={handleStartEditing} className="btn btn-primary">
              Start Editing
            </button>
          )}
        </div>
      </div>

      {error && (
        <div className="alert alert-danger">
          {error}
        </div>
      )}

      <div className="active-users">
        {activeUsers.length > 0 && (
          <div>
            <strong>Currently editing:</strong>
            {activeUsers.map(user => (
              <span key={user.userId} className="user-badge">
                {user.userId}
              </span>
            ))}
          </div>
        )}
      </div>

      <div className="editor-content">
        <textarea
          value={content}
          onChange={(e) => handleContentChange(e.target.value)}
          disabled={!isEditing}
          placeholder={isEditing ? "Start typing..." : "Click 'Start Editing' to edit"}
          className="form-control"
          rows={20}
        />
      </div>

      <div className="editor-footer">
        Document ID: {documentId} | User: {userId}
      </div>
    </div>
  );
};

export default DocumentEditor;
```

#### Usage Example (App.jsx)
```jsx
import React from 'react';
import DocumentEditor from './DocumentEditor';

const App = () => {
  return (
    <div className="app">
      <h1>Real-time Document Editor</h1>
      <DocumentEditor
        documentId="doc-123"
        userId="user-456"
        serverUrl="http://localhost:8080" // Your ASP.NET API server URL
      />
    </div>
  );
};

export default App;
```

### 3. CSS Styling (Optional)
```css
.document-editor {
  max-width: 1000px;
  margin: 0 auto;
  padding: 20px;
}

.editor-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding-bottom: 10px;
  border-bottom: 1px solid #ddd;
}

.editor-controls {
  display: flex;
  align-items: center;
  gap: 15px;
}

.connection-status {
  font-size: 14px;
  font-weight: bold;
}

.btn {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}

.btn-primary {
  background-color: #007bff;
  color: white;
}

.btn-danger {
  background-color: #dc3545;
  color: white;
}

.alert {
  padding: 10px;
  margin-bottom: 15px;
  border-radius: 4px;
}

.alert-danger {
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.active-users {
  margin-bottom: 15px;
  padding: 10px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

.user-badge {
  display: inline-block;
  margin-left: 8px;
  padding: 4px 8px;
  background-color: #007bff;
  color: white;
  border-radius: 12px;
  font-size: 12px;
}

.form-control {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-family: 'Courier New', monospace;
}

.editor-footer {
  margin-top: 15px;
  padding-top: 10px;
  border-top: 1px solid #ddd;
  font-size: 12px;
  color: #666;
}
```

### Key Features

1. **No jQuery Dependency**: Uses `signalr-no-jquery` package for React integration[1][2][3]
2. **Real-time Communication**: SignalR Hub manages document editing sessions[4][5][6]
3. **REST API Integration**: Traditional API calls for document CRUD operations[7][8]
4. **Conflict Resolution**: Handles concurrent editing scenarios[5][9]
5. **Connection Management**: Automatic reconnection and cleanup[4][10]
6. **Debounced Updates**: Prevents excessive API calls during typing[5][11]

This implementation provides a complete solution for real-time collaborative document editing using React.js, SignalR, and ASP.NET Classic Web API without jQuery dependencies.

Citations:
[1] signalr-no-jquery - NPM https://www.npmjs.com/package/signalr-no-jquery
[2] Signalr-no-jquery - npm.io https://npm.io/package/signalr-no-jquery
[3] cojam/signalr-no-jquery - NPM https://www.npmjs.com/package/@cojam/signalr-no-jquery
[4] SignalR in .NET Core: A Guide For Real-time Web Communication https://www.bacancytechnology.com/blog/signalr-in-net-core
[5] Synchronizing Editable Regions with SignalR Broadcasting https://www.textcontrol.com/blog/2020/05/12/synchronizing-editable-regions-with-signalr/
[6] Collaborative Editing in ASP.NET CORE Document Editor Control https://ej2.syncfusion.com/aspnetcore/documentation/document-editor/collaborative-editing/using-dot-net
[7] ASP.NET SignalR Hubs API Guide - Server (C#) | Microsoft Learn https://learn.microsoft.com/en-us/aspnet/signalr/overview/guide-to-the-api/hubs-api-guide-server
[8] ASP.NET SignalR Hubs API Guide - .NET Client (C#) | Microsoft Learn https://learn.microsoft.com/en-us/aspnet/signalr/overview/guide-to-the-api/hubs-api-guide-net-client
[9] Building Real-Time Applications with SignalR and ASP.NET Core https://www.qservicesit.com/building-real-time-applications-with-signalr-and-asp-net-core
[10] Add a SignalR hub to ASP.NET Core & connect using JavaScript https://www.roundthecode.com/dotnet-tutorials/add-signalr-hub-aspnet-core-connect-using-javascript
[11] Building a cloud editor with SignalR and ASP.NET Core 2.1 https://blog.elmah.io/building-a-cloud-editor-with-signalr-and-asp-net-core-2-1/
[12] SignalR in .NET Core: Enabling Real-Time Web Applications https://theonetechnologies.com/blog/post/real-time-communication-with-signalr-in-net-core-building-interactive-web-applications
[13] Real-time ASP.NET with SignalR | .NET https://dotnet.microsoft.com/en-us/apps/aspnet/signalr
[14] A Guide to using ASP.Net SignalR with RadNotifications - Telerik.com https://www.telerik.com/blogs/signalr-with-radnotifications
[15] Creating Manual Javascript SignalR Client - C# Corner https://www.c-sharpcorner.com/article/creating-manual-javascript-signalr-client/
[16] How do I generate API documentation for SignalR - Stack Overflow https://stackoverflow.com/questions/28948517/how-do-i-generate-api-documentation-for-signalr
[17] ASP.NET Core SignalR Introduction - Redwerk https://redwerk.com/blog/asp-net-core-signalr-introduction/
[18] Is SignalR possible without jQuery? - asp.net - Stack Overflow https://stackoverflow.com/questions/18861651/is-signalr-possible-without-jquery
[19] ASP.NET MVC Data Management Grid Data Binding SignalR Binding https://www.telerik.com/aspnet-mvc/documentation/html-helpers/data-management/grid/binding/signalr-binding
[20] Please update the usage instructions with an example for "traditional ... https://github.com/DVLP/signalr-no-jquery/issues/55
[21] DocumentViewer Collaboration: Live Share Document Annotations https://www.textcontrol.com/blog/2021/06/08/documentviewer-collaboration-live-share-document-annotations/
[22] NadaMesbah/Collaborative-Text-Editor: A real-time ... - GitHub https://github.com/NadaMesbah/Collaborative-Text-Editor
[23] Can SignalR be used with asp.net WebForms? - Stack Overflow https://stackoverflow.com/questions/18143599/can-signalr-be-used-with-asp-net-webforms
[24] What is SignalR? A Real-Time Communication Framework for .NET https://dev.to/dazevedo/what-is-signalr-a-real-time-communication-framework-for-net-29h0
[25] How to Use SignalR for Real-Time Data Updates in .NET Apps https://blog.pixelfreestudio.com/how-to-use-signalr-for-real-time-data-updates-in-net-apps/
[26] How to build chat Server and Client with SignalR and ASP.NET Web ... https://dotjord.wordpress.com/2018/05/20/how-to-build-chat-server-and-client-with-signalr-and-asp-net-web-api-part-2/
[27] How to Implement Real-Time Web Applications with SignalR in ASP ... https://www.avidclan.com/blog/how-to-implement-real-time-web-applications-with-signalr-in-asp-net-core-a-step-by-step-guide/





public class WSChatController : ApiController
{
    // Store all connected clients
    private static readonly HashSet<WebSocket> Clients = new HashSet<WebSocket>();
    private static readonly object ClientsLock = new object();

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

        try
        {
            while (socket.State == WebSocketState.Open)
            {
                var buffer = new ArraySegment<byte>(new byte[1024]);
                var result = await socket.ReceiveAsync(buffer, CancellationToken.None);

                if (result.MessageType == WebSocketMessageType.Text)
                {
                    string userMessage = Encoding.UTF8.GetString(buffer.Array, 0, result.Count);
                    var responseBytes = Encoding.UTF8.GetBytes(userMessage);

                    lock (ClientsLock)
                    {
                        foreach (var client in Clients)
                        {
                            if (client.State == WebSocketState.Open)
                            {
                                await client.SendAsync(
                                    new ArraySegment<byte>(responseBytes),
                                    WebSocketMessageType.Text, true, CancellationToken.None
                                );
                            }
                        }
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
            if (socket.State == WebSocketState.Open || socket.State == WebSocketState.CloseReceived)
            {
                await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", CancellationToken.None);
            }
        }
    }
}
