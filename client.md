<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Collaborative Editor</title>
  <style>
    #messages { border: 1px solid #ccc; height: 180px; overflow-y: scroll; margin-bottom: 1em; padding: 10px; }
    #edit-status { font-weight: bold; color: #c0392b; margin-bottom: 1em; }
    .editing { background: #e0ffdc; }
  </style>
</head>
<body>
<h2>Collaborative Editor</h2>

<div>
  <label>
    Document ID: 
    <input type="text" id="docIdInput" value="doc1" size="10" />
    <span id="edit-status"></span>
  </label>
</div>
<textarea id="editor" rows="6" cols="70" placeholder="Type here..." disabled></textarea>
<br/>
<button id="editBtn">Start Editing</button>
<button id="stopBtn" disabled>Stop Editing</button>

<div id="messages"></div>

<script>
  const docIdInput = document.getElementById('docIdInput');
  const editor = document.getElementById('editor');
  const editStatus = document.getElementById('edit-status');
  const editBtn = document.getElementById('editBtn');
  const stopBtn = document.getElementById('stopBtn');
  const messages = document.getElementById('messages');
  const wsUrl = 'wss://localhost:44375/api/WSChat';

  let isEditing = false;
  let myUserId = prompt("Enter your user ID") || 'guest_' + Math.floor(Math.random() * 1000);
  let socket;
  let currentlyEditedDocs = {};

  function logMessage(msg) {
    let div = document.createElement('div');
    div.textContent = msg;
    messages.appendChild(div);
    messages.scrollTop = messages.scrollHeight;
  }

  function updateEditStatus(docId, editingUser) {
    if (!editingUser) {
      editStatus.textContent = 'Available';
      editStatus.style.color = 'green';
      editor.disabled = !isEditing;
      editor.classList.remove('editing');
    } else if (editingUser === myUserId) {
      editStatus.textContent = '(You are editing)';
      editStatus.style.color = 'blue';
      editor.disabled = false;
      editor.classList.add('editing');
    } else {
      editStatus.textContent = `Being edited by: ${editingUser}`;
      editStatus.style.color = '#c0392b';
      editor.disabled = true;
      editor.classList.remove('editing');
    }
  }

  function connect() {
    socket = new WebSocket(wsUrl);

    socket.onopen = function() {
      socket.send('userId:' + myUserId);
      
      logMessage('Connected as ' + myUserId);
    };

    socket.onmessage = function(event) {
      logMessage(event.data);

      const docId = docIdInput.value.trim();

      // Special: Deny editing if already taken
      if (event.data.startsWith('Document ' + docId + ' is already being edited by ')) {
        editStatus.textContent = event.data;
        editStatus.style.color = "#c0392b";
        editor.disabled = true;
        editBtn.disabled = false;
        stopBtn.disabled = true;
        isEditing = false;
        return;
      }

      // For user edit start/stop
      if (event.data.startsWith('User ') && event.data.includes('started editing document')) {
        let match = event.data.match(/^User (.+?) started editing document (\S+)/);
        if (match && match[2] === docId) {
          let editingUser = match[1];
          currentlyEditedDocs[docId] = editingUser;
          updateEditStatus(docId, editingUser);
        }
      } else if (event.data.startsWith('User ') && event.data.includes('stopped editing document')) {
        let match = event.data.match(/^User (.+?) stopped editing document (\S+)/);
        if (match && match[2] === docId) {
          currentlyEditedDocs[docId] = null;
          updateEditStatus(docId, null);
        }
      } else if (event.data.startsWith('User ') && event.data.includes('disconnected and stopped editing document')) {
        let match = event.data.match(/^User (.+?) disconnected and stopped editing document (\S+)/);
        if (match && match[2] === docId) {
          currentlyEditedDocs[docId] = null;
          updateEditStatus(docId, null);
        }
      }
    };

    socket.onclose = function() {
      logMessage('Disconnected.');
      editBtn.disabled = false;
      stopBtn.disabled = true;
      editor.disabled = true;
    };

    socket.onerror = function() {
      logMessage('WebSocket error.');
    };
  }

  editBtn.onclick = function() {
    if (!socket || socket.readyState !== WebSocket.OPEN) return;
    let docId = docIdInput.value.trim();
    socket.send('edit:start:' + docId);
    isEditing = true;
    editBtn.disabled = true;
    stopBtn.disabled = false;
  };

  stopBtn.onclick = function() {
    if (!socket || socket.readyState !== WebSocket.OPEN) return;
    let docId = docIdInput.value.trim();
    socket.send('edit:stop:' + docId);
    isEditing = false;
    editBtn.disabled = false;
    stopBtn.disabled = true;
    editor.disabled = true;
    editor.classList.remove('editing');
  };

  docIdInput.onchange = function() {
    isEditing = false;
    editBtn.disabled = false;
    stopBtn.disabled = true;
    editor.disabled = true;
    editor.classList.remove('editing');
    // Show status for new doc
    updateEditStatus(docIdInput.value.trim(), currentlyEditedDocs[docIdInput.value.trim()]);
  };

  editor.onkeydown = function(e) {
    if (!isEditing && editor.disabled) e.preventDefault();
  };

  connect();
</script>
</body>
</html>
