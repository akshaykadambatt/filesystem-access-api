# filesystem-access-api (A image browser page)
 filesystem-access-api
 
The File System Access API: simplifying access to local files
The File System Access API allows web apps to read or save changes directly to files and folders on the user's device.
What is the File System Access API? #
The File System Access API (formerly known as Native File System API and prior to that it was called Writeable Files API) enables developers to build powerful web apps that interact with files on the user's local device, like IDEs, photo and video editors, text editors, and more. After a user grants a web app access, this API allows them to read or save changes directly to files and folders on the user's device. Beyond reading and writing files, the File System Access API provides the ability to open a directory and enumerate its contents.

If you've worked with reading and writing files before, much of what I'm about to share will be familiar to you. I encourage you to read it anyway, because not all systems are alike.

Using the File System Access API #
To show off the power and usefulness of the File System Access API, I wrote a single file text editor. It lets you open a text file, edit it, save the changes back to disk, or start a new file and save the changes to disk. It's nothing fancy, but provides enough to help you understand the concepts.

Ask the user to pick a file to read #
The entry point to the File System Access API is window.showOpenFilePicker(). When called, it shows a file picker dialog box, and prompts the user to select a file. After they select a file, the API returns an array of file handles. An optional options parameter lets you influence the behavior of the file picker, for example, by allowing the user to select multiple files, or directories, or different file types. Without any options specified, the file picker allows the user to select a single file. This is perfect for a text editor.

Like many other powerful APIs, calling showOpenFilePicker() must be done in a secure context, and must be called from within a user gesture.


let fileHandle;
butOpenFile.addEventListener('click', async () => {
  // Destructure the one-element array.
  [fileHandle] = await window.showOpenFilePicker();
  // Do something with the file handle.
});
Once the user selects a file, showOpenFilePicker() returns an array of handles, in this case a one-element array with one FileSystemFileHandle that contains the properties and methods needed to interact with the file.

It's helpful to keep a reference to the file handle around so that it can be used later. It'll be needed to save changes back to the file, or to perform any other file operations.

Read a file from the file system #
Now that you have a handle to a file, you can get the file's properties, or access the file itself. For now, I'll simply read its contents. Calling handle.getFile() returns a File object, which contains a blob. To get the data from the blob, call one of its methods (slice(), stream(), text(), arrayBuffer()).


const file = await fileHandle.getFile();
const contents = await file.text();
The File object returned by FileSystemFileHandle.getFile() is only readable as long as the underlying file on disk hasn't changed. If the file on disk is modified, the File object becomes unreadable and you'll need to call getFile() again to get a new File object to read the changed data.

Putting it all together #
When users click the Open button, the browser shows a file picker. Once they've selected a file, the app reads the contents and puts them into a <textarea>.


let fileHandle;
butOpenFile.addEventListener('click', async () => {
  [fileHandle] = await window.showOpenFilePicker();
  const file = await fileHandle.getFile();
  const contents = await file.text();
  textArea.value = contents;
});
Write the file to the local file system #
In the text editor, there are two ways to save a file: Save, and Save As. Save simply writes the changes back to the original file using the file handle retrieved earlier. But Save As creates a new file, and thus requires a new file handle.

Create a new file #
To save a file, call showSaveFilePicker(), which will show the file picker in "save" mode, allowing the user to pick a new file they want to use for saving. For the text editor, I also wanted it to automatically add a .txt extension, so I provided some additional parameters.


async function getNewFileHandle() {
  const options = {
    types: [
      {
        description: 'Text Files',
        accept: {
          'text/plain': ['.txt'],
        },
      },
    ],
  };
  const handle = await window.showSaveFilePicker(options);
  return handle;
}
Save changes to disk #
You can find all the code for saving changes to a file in my text editor demo on GitHub. The core file system interactions are in fs-helpers.js. At its simplest, the process looks like the code below. I'll walk through each step and explain it.


async function writeFile(fileHandle, contents) {
  // Create a FileSystemWritableFileStream to write to.
  const writable = await fileHandle.createWritable();
  // Write the contents of the file to the stream.
  await writable.write(contents);
  // Close the file and write the contents to disk.
  await writable.close();
}
Writing data to disk uses a FileSystemWritableFileStream object, essentially a WritableStream. Create the stream by calling createWritable() on the file handle object. When createWritable() is called, the browser first checks if the user has granted write permission to the file. If permission to write hasn't been granted, the browser will prompt the user for permission. If permission isn't granted, createWritable() will throw a DOMException, and the app will not be able to write to the file. In the text editor, these DOMExceptions are handled in the saveFile() method.

The write() method takes a string, which is what's needed for a text editor. But it can also take a BufferSource, or a Blob. For example, you can pipe a stream directly to it:


async function writeURLToFile(fileHandle, url) {
  // Create a FileSystemWritableFileStream to write to.
  const writable = await fileHandle.createWritable();
  // Make an HTTP request for the contents.
  const response = await fetch(url);
  // Stream the response into the file.
  await response.body.pipeTo(writable);
  // pipeTo() closes the destination pipe by default, no need to close it.
}
You can also seek(), or truncate() within the stream to update the file at a specific position, or resize the file.

Caution: Changes are not written to disk until the stream is closed, either by calling close() or when the stream is automatically closed by the pipe.

Storing file handles in IndexedDB #
File handles are serializable, which means that you can save a file handle to IndexedDB, or call postMessage() to send them between the same top-level origin.

Saving file handles to IndexedDB means that you can store state, or remember which files a user was working on. This makes it possible to keep a list of recently opened or edited files, offer to re-open the last file when the app is opened, etc. In the text editor, I store a list of the five most recent files the user has opened, making it easy to access those files again.

The code example below shows storing and retrieving a file handle. You can see this in action over on Glitch (I use the idb-keyval library for brevity).


import { get, set } from 'https://unpkg.com/idb-keyval@5.0.2/dist/esm/index.js';

const pre = document.querySelector('pre');
const button = document.querySelector('button');

button.addEventListener('click', async () => {
  try {
    // Try retrieving the file handle.
    const fileHandleOrUndefined = await get('file');    
    if (fileHandleOrUndefined) {      
      pre.textContent =
          `Retrieved file handle "${fileHandleOrUndefined.name}" from IndexedDB.`;
      return;
    }
    // This always returns an array, but we just need the first entry.
    const [fileHandle] = await window.showOpenFilePicker();
    // Store the file handle.
    await set('file', fileHandle);    
    pre.textContent =
        `Stored file handle for "${fileHandle.name}" in IndexedDB.`;
  } catch (error) {
    alert(error.name, error.message);
  }
});
Since permissions currently are not persisted between sessions, you should verify whether the user has granted permission to the file using queryPermission(). If they haven't, use requestPermission() to (re-)request it.

In the text editor, I created a verifyPermission() method that checks if the user has already granted permission, and if required, makes the request.


async function verifyPermission(fileHandle, readWrite) {
  const options = {};
  if (readWrite) {
    options.mode = 'readwrite';
  }
  // Check if permission was already granted. If so, return true.
  if ((await fileHandle.queryPermission(options)) === 'granted') {
    return true;
  }
  // Request permission. If the user grants permission, return true.
  if ((await fileHandle.requestPermission(options)) === 'granted') {
    return true;
  }
  // The user didn't grant permission, so return false.
  return false;
}
By requesting write permission with the read request, I reduced the number of permission prompts: the user sees one prompt when opening the file, and grants permission to both read and write to it.

Opening a directory and enumerating its contents #
To enumerate all files in a directory, call showDirectoryPicker(). The user selects a directory in a picker, after which a FileSystemDirectoryHandle is returned, which lets you enumerate and access the directory's files.


const butDir = document.getElementById('butDirectory');
butDir.addEventListener('click', async () => {
  const dirHandle = await window.showDirectoryPicker();
  for await (const entry of dirHandle.values()) {
    console.log(entry.kind, entry.name);
  }
});
Creating or accessing files and folders in a directory #
From a directory, you can create or access files and folders using the getFileHandle() or respectively the getDirectoryHandle() method. By passing in an optional options object with a key of create and a boolean value of true or false, you can determine if a new file or folder should be created if it doesn't exist.


// In an existing directory, create a new directory named "My Documents".
const newDirectoryHandle = await existingDirectoryHandle.getDirectoryHandle('My Documents', {
  create: true,
});
// In this new directory, create a file named "My Notes.txt".
const newFileHandle = await newDirectoryHandle.getFileHandle('My Notes.txt', { create: true });
Resolving the path of an item in a directory #
When working with files or folders in a directory, it can be useful to resolve the path of the item in question. This can be done with the aptly named resolve() method. For resolving, the item can be a direct or indirect child of the directory.


// Resolve the path of the previously created file called "My Notes.txt".
const path = await newDirectoryHandle.resolve(newFileHandle);
// `path` is now ["My Documents", "My Notes.txt"]
Deleting files and folders in a directory #
If you have obtained access to a directory, you can delete the contained files and folders with the removeEntry() method. For folders, deletion can optionally be recursive and include all subfolders and the therein contained files.


// Delete a file.
await directoryHandle.removeEntry('Abandoned Masterplan.txt');
// Recursively delete a folder.
await directoryHandle.removeEntry('Old Stuff', { recursive: true });
Drag and drop integration #
The HTML Drag and Drop interfaces enable web applications to accept dragged and dropped files on a web page. During a drag and drop operation, dragged file and directory items are associated with file entries and directory entries respectively. The DataTransferItem.getAsFileSystemHandle() method returns a promise with a FileSystemFileHandle object if the dragged item is a file, and a promise with a FileSystemDirectoryHandle object if the dragged item is a directory. The listing below shows this in action. Note that the Drag and Drop interface's DataTransferItem.kind will be "file" for both files and directories, whereas the File System Access API's FileSystemHandle.kind will be "file" for files and "directory" for directories.


elem.addEventListener('dragover', (e) => {
  // Prevent navigation.
  e.preventDefault();
});

elem.addEventListener('drop', async (e) => {
  // Prevent navigation.
  e.preventDefault();
  // Process all of the items.
  for (const item of e.dataTransfer.items) {
    // Careful: `kind` will be 'file' for both file
    // _and_ directory entries.
    if (item.kind === 'file') {
      const entry = await item.getAsFileSystemHandle();
      if (entry.kind === 'directory') {
        handleDirectoryEntry(entry);
      } else {
        handleFileEntry(entry);
      }
    }
  }
});
Accessing the origin-private file system #
The origin-private file system is a storage endpoint that, as the name suggests, is private to the origin of the page. While browsers will typically implement this by persisting the contents of this origin-private file system to disk somewhere, it is not intended that the contents be easily user accessible. Similarly, there is no expectation that files or directories with names matching the names of children of the origin-private file system exist. While the browser might make it seem that there are files, internally—since this is an origin-private file system—the browser might store these "files" in a database or any other data structure. Essentially: what you create with this API, do not expect to find it 1:1 somewhere on the hard disk. You can operate as usual on the origin-private file system once you have access to the root FileSystemDirectoryHandle.


const root = await navigator.storage.getDirectory();
// Create a new file handle.
const fileHandle = await root.getFileHandle('Untitled.txt', { create: true });
// Create a new directory handle.
const dirHandle = await root.getDirectoryHandle('New Folder', { create: true });
// Recursively remove a directory.
await root.removeEntry('Old Stuff', { recursive: true });
Polyfilling #
It is not possible to completely polyfill the File System Access API methods.

The showOpenFilePicker() method can be approximated with an <input type="file"> element.
The showSaveFilePicker() method can be simulated with a <a download="file_name"> element, albeit this will trigger a programmatic download and not allow for overwriting existing files.
The showDirectoryPicker() method can be somewhat emulated with the non-standard <input type="file" webkitdirectory> element.
We have developed a library called browser-fs-access that uses the File System Access API wherever possible and that falls back to these next best options in all other cases.
