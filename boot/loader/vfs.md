## vfs

## NodeConstructor
## Node()

```
Node::Node()
	:
	fRefCount(1)
{
}
```

## DirectoryConstructor
## Directory ()

> Inherited from [Node](#NodeConstructor)

```

Directory::Directory()
	: Node()
{
}

```
* [Node()](#NodeConstructor) constructor

## Lookup

> arg1 = const char* name

> arg2 = bool traverseLinks

> return [Node](#NodeConstructor)*

```
Node*
Directory::Lookup(const char* name, bool traverseLinks)
{
	Node* node = LookupDontTraverse(name);
	if (node == NULL)
		return NULL;

	if (!traverseLinks || !S_ISLNK(node->Type()))
		return node;

	// the node is a symbolic link, so we have to resolve the path
	char linkPath[B_PATH_NAME_LENGTH];
	status_t error = node->ReadLink(linkPath, sizeof(linkPath));

	node->Release();
		// we don't need this one anymore

	if (error != B_OK)
		return NULL;

	// let open_from() do the real work
	int fd = open_from(this, linkPath, O_RDONLY);
	if (fd < 0)
		return NULL;

	node = get_node_from(fd);
	if (node != NULL)
		node->Acquire();

	close(fd);
	return node;
}
```
* Node*
* Directory::Lookup(const char* name, bool traverseLinks)
* {	// name = system
* 	// traverseLinks = true

* // LookupDontTraverse returns the node of that path.
* [Node](#NodeConstructor)* node = LookupDontTraverse(name);	// LookupDontTraverse is defined in the filesytem files. src/system/boot/loader/file_systems/bfs/Directory.cpp
* if (node == NULL)	// if that node is not existed in the filesytem
	* return NULL;

* if (!traverseLinks || !S_ISLNK(node->Type()))
	* return node;

* // the node is a symbolic link, so we have to resolve the path
* char linkPath[B_PATH_NAME_LENGTH];
* status_t error = node->ReadLink(linkPath, sizeof(linkPath));

* node->Release();
* // we don't need this one anymore

* if (error != B_OK)
	* return NULL;

* // let open_from() do the real work
* int fd = open_from(this, linkPath, O_RDONLY);
* if (fd < 0)
	* return NULL;

* node = get_node_from(fd);
* if (node != NULL)
	* node->Acquire();

* close(fd);
* return node;
* }
---


## vfs_init(stage2_args *args)


```
status_t
vfs_init(stage2_args *args)
{
	gRoot = new(nothrow) RootFileSystem();
	if (gRoot == NULL)
		return B_NO_MEMORY;

	return B_OK;
}
```

## get_boot_file_system

* arg1 = stage2_args*
* arg2 = [BootVolume](#BootVolume)&


```

/*! Gets the boot device, scans all of its partitions, gets the
	boot partition, and mounts its file system.
	\param args The stage 2 arguments.
	\param _bootVolume On success set to the boot volume.
	\return \c B_OK on success, another error code otherwise.
*/
status_t
get_boot_file_system(stage2_args* args, BootVolume& _bootVolume)
{
	status_t error = platform_add_boot_device(args, &gBootDevices);
	if (error != B_OK)
		return error;

	NodeIterator iterator = gBootDevices.GetIterator();
	while (iterator.HasNext()) {
		Node *device = iterator.Next();

		error = add_partitions_for(device, false, true);
		if (error != B_OK)
			continue;

		NodeList bootPartitions;
		error = platform_get_boot_partitions(args, device, &gPartitions, &bootPartitions);
		if (error != B_OK)
			continue;

		NodeIterator partitionIterator = bootPartitions.GetIterator();
		while (partitionIterator.HasNext()) {
			Partition *partition = (Partition*)partitionIterator.Next();

			Directory *fileSystem;
			error = partition->Mount(&fileSystem, true);
			if (error != B_OK) {
				// this partition doesn't contain any known file system; we
				// don't need it anymore
				gPartitions.Remove(partition);
				delete partition;
				continue;
			}

			// init the BootVolume
			error = _bootVolume.SetTo(fileSystem);
			if (error != B_OK)
				continue;

			sBootDevice = device;
			return B_OK;
		}
	}

	return B_ERROR;
}

```
* [platform_add_boot_device](/boot/efi/devices.md#platform_add_boot_device)
* error = _bootVolume.SetTo(fileSystem);	// [SetTo](#SetTo)



## BootVolume()

```
BootVolume::BootVolume()
	:
	fRootDirectory(NULL),
	fSystemDirectory(NULL),
	fPackageVolumeInfo(NULL),
	fPackageVolumeState(NULL)
{
}
```

## SetTo
## BootVolume::SetTo

```
status_t
BootVolume::SetTo(Directory* rootDirectory,
	PackageVolumeInfo* packageVolumeInfo,
	PackageVolumeState* packageVolumeState)
{
	Unset();

	status_t error = _SetTo(rootDirectory, packageVolumeInfo,
		packageVolumeState);
	if (error != B_OK)
		Unset();

	return error;
}

```

* status_t error = [_SetTo](#_SetTo)(rootDirectory, packageVolumeInfo, packageVolumeState);

## _SetTo
## BootVolume::_SetTo

```
status_t
BootVolume::_SetTo(Directory* rootDirectory,
	PackageVolumeInfo* packageVolumeInfo,
	PackageVolumeState* packageVolumeState)
{
	Unset();

	if (rootDirectory == NULL)
		return B_BAD_VALUE;

	fRootDirectory = rootDirectory;
	fRootDirectory->Acquire();

	// find the system directory
	Node* systemNode = fRootDirectory->Lookup("system", true);
	if (systemNode == NULL || !S_ISDIR(systemNode->Type())) {
		if (systemNode != NULL)
			systemNode->Release();
		Unset();
		return B_ENTRY_NOT_FOUND;
	}

	fSystemDirectory = static_cast<Directory*>(systemNode);

	if (packageVolumeInfo == NULL) {
		// get a package volume info
		BReference<PackageVolumeInfo> packageVolumeInfoReference(
			new(std::nothrow) PackageVolumeInfo);
		status_t error = packageVolumeInfoReference->SetTo(fSystemDirectory,
			"packages");
		if (error != B_OK) {
			// apparently not packaged
			return B_OK;
		}

		fPackageVolumeInfo = packageVolumeInfoReference.Detach();
	} else {
		fPackageVolumeInfo = packageVolumeInfo;
		fPackageVolumeInfo->AcquireReference();
	}

	fPackageVolumeState = packageVolumeState != NULL
		? packageVolumeState : fPackageVolumeInfo->States().Head();

	// try opening the system package
	int packageFD = _OpenSystemPackage();
	if (packageFD < 0)
		return packageFD;

	// mount packagefs
	Directory* packageRootDirectory;
	status_t error = packagefs_mount_file(packageFD, fSystemDirectory,
		packageRootDirectory);
	close(packageFD);
	if (error != B_OK) {
		Unset();
		return error;
	}

	fSystemDirectory->Release();
	fSystemDirectory = packageRootDirectory;

	return B_OK;
}

```


## mount_file_systems

> arg1 = stage2_args *args

```
/** Mounts all file systems recognized on the given device by
 *	calling the add_partitions_for() function on them.
 */

status_t
mount_file_systems(stage2_args *args)
{
	// mount other partitions on boot device (if any)
	NodeIterator iterator = gPartitions.GetIterator();

	Partition *partition = NULL;
	while ((partition = (Partition *)iterator.Next()) != NULL) {
		// don't scan known partitions again
		if (partition->IsFileSystem())
			continue;

		// remove the partition if it doesn't contain a (known) file system
		if (partition->Scan(true) != B_OK && !partition->IsFileSystem()) {
			gPartitions.Remove(partition);
			delete partition;
		}
	}

	// add all block devices the platform has for us

	status_t status = platform_add_block_devices(args, &gBootDevices);
	if (status < B_OK)
		return status;

	iterator = gBootDevices.GetIterator();
	Node *device = NULL, *last = NULL;
	while ((device = iterator.Next()) != NULL) {
		// don't scan former boot device again
		if (device == sBootDevice)
			continue;

		if (add_partitions_for(device, true) == B_OK) {
			// ToDo: we can't delete the object here, because it must
			//	be removed from the list before we know that it was
			//	deleted.

/*			// if the Release() deletes the object, we need to skip it
			if (device->Release() > 0) {
				list_remove_item(&gBootDevices, device);
				device = last;
			}
*/
(void)last;
		}
		last = device;
	}

	if (gPartitions.IsEmpty())
		return B_ENTRY_NOT_FOUND;

#if 0
	void *cookie;
	if (gRoot->Open(&cookie, O_RDONLY) == B_OK) {
		Directory *directory;
		while (gRoot->GetNextNode(cookie, (Node **)&directory) == B_OK) {
			char name[256];
			if (directory->GetName(name, sizeof(name)) == B_OK)
				printf(":: %s (%p)\n", name, directory);

			void *subCookie;
			if (directory->Open(&subCookie, O_RDONLY) == B_OK) {
				while (directory->GetNextEntry(subCookie, name, sizeof(name)) == B_OK) {
					printf("\t%s\n", name);
				}
				directory->Close(subCookie);
			}
		}
		gRoot->Close(cookie);
	}
#endif

	return B_OK;
}
```


## open_from

> arg1 = [Directory]() *directory

> arg2 = const char *name

> arg3 = [mode_t]() permissions

```
int
open_from(Directory *directory, const char *name, int mode, mode_t permissions)
{
	if (name[0] == '/') {
		// ignore the directory and start from root if we are asked to do that
		directory = gRoot;
		name++;
	}

	char* path = (char*)malloc(B_PATH_NAME_LENGTH);
	if (path == NULL)
		return B_NO_MEMORY;

	if (strlcpy(path, name, B_PATH_NAME_LENGTH) >= B_PATH_NAME_LENGTH) {
		free(path);
		return B_NAME_TOO_LONG;
	}

	Node *node;
	status_t error = get_node_for_path(directory, path, &node);
	if (error != B_OK) {
		if (error != B_ENTRY_NOT_FOUND) {
			free(path);
			return error;
		}

		if ((mode & O_CREAT) == 0) {
			free(path);
			return B_ENTRY_NOT_FOUND;
		}

		// try to resolve the parent directory
		strlcpy(path, name, B_PATH_NAME_LENGTH);
		if (char* lastSlash = strrchr(path, '/')) {
			if (lastSlash[1] == '\0') {
				free(path);
				return B_ENTRY_NOT_FOUND;
			}

			*lastSlash = '\0';
			name = lastSlash + 1;

			// resolve the directory
			if (get_node_for_path(directory, path, &node) != B_OK) {
				free(path);
				return B_ENTRY_NOT_FOUND;
			}

			if (node->Type() != S_IFDIR) {
				node->Release();
				free(path);
				return B_NOT_A_DIRECTORY;
			}

			directory = static_cast<Directory*>(node);
		} else
			directory->Acquire();

		// create the file
		error = directory->CreateFile(name, permissions, &node);
		directory->Release();

		if (error != B_OK) {
			free(path);
			return error;
		}
	} else if ((mode & O_EXCL) != 0) {
		node->Release();
		free(path);
		return B_FILE_EXISTS;
	}

	int fd = open_node(node, mode);

	node->Release();
	free(path);
	return fd;
}
```
+ int
+ open_from(Directory *directory, const char *name, int mode, mode_t permissions)
+ {
	+ if (name[0] == '/') {
		+ // if the received path`s first letter is '/' then start from the root.
		+ // ignore the directory and start from root if we are asked to do that
		+ directory = gRoot;	// let the directory to point to root.
		+ name++;
	+ }

	+ char* path = (char*)malloc(B_PATH_NAME_LENGTH);	// reserve some space for the path string.
	+ if (path == NULL)	// if path is not reserved.
		+ return B_NO_MEMORY;	// just return from this function error.

	+ if (strlcpy(path, name, B_PATH_NAME_LENGTH) >= B_PATH_NAME_LENGTH) {	// copy the received path to the above created string path.
		+ free(path);	// free the allocated path
		+ return B_NAME_TOO_LONG;	// return from the function error
	+ }

	+ [Node](#NodeConstructor) *node;	// node object
	+ status_t error = [get_node_for_path](#get_node_for_path)(directory, path, &node);
	+ if (error != B_OK) {
		+ if (error != B_ENTRY_NOT_FOUND) {
			+ free(path);
			+ return error;
		+ }

		+ if ((mode & O_CREAT) == 0) {
			+ free(path);
			+ return B_ENTRY_NOT_FOUND;
		+ }

		+ // try to resolve the parent directory
		+ strlcpy(path, name, B_PATH_NAME_LENGTH);
		+ if (char* lastSlash = strrchr(path, '/')) {
			+ if (lastSlash[1] == '\0') {
				+ free(path);
				+ return B_ENTRY_NOT_FOUND;
			+ }

			+ *lastSlash = '\0';
			+ name = lastSlash + 1;

			+ // resolve the directory
			+ if (get_node_for_path(directory, path, &node) != B_OK) {
				+ free(path);
				+ return B_ENTRY_NOT_FOUND;
			+ }

			+ if (node->Type() != S_IFDIR) {
				+ node->Release();
				+ free(path);
				+ return B_NOT_A_DIRECTORY;
			+ }

			+ directory = static_cast<Directory*>(node);
		+ } else
			+ directory->Acquire();

		+ // create the file
		+ error = directory->CreateFile(name, permissions, &node);
		+ directory->Release();

		+ if (error != B_OK) {
			+ free(path);
			+ return error;
		+ }
	+ } else if ((mode & O_EXCL) != 0) {
		+ node->Release();
		+ free(path);
		+ return B_FILE_EXISTS;
	+ }

	+ int fd = open_node(node, mode);

	+ node->Release();
	+ free(path);
	+ return fd;
+ }

## get_node_for_path

> arg1 = Directory *directory

> arg2 = char *path

> arg3 = Node **_node

```
/*!	Resolves \a directory + \a path to a node.
	Note that \a path will be modified by the function.
*/
static status_t
get_node_for_path(Directory *directory, char *path, Node **_node)
{
	directory->Acquire();
		// balance Acquire()/Release() calls

	while (true) {
		Node *nextNode;
		char *nextPath;

		// walk to find the next path component ("path" will point to a single
		// path component), and filter out multiple slashes
		for (nextPath = path + 1; nextPath[0] != '\0' && nextPath[0] != '/'; nextPath++);

		if (*nextPath == '/') {
			*nextPath = '\0';
			do
				nextPath++;
			while (*nextPath == '/');
		}

		nextNode = directory->Lookup(path, true);
		directory->Release();

		if (nextNode == NULL)
			return B_ENTRY_NOT_FOUND;

		path = nextPath;
		if (S_ISDIR(nextNode->Type()))
			directory = (Directory *)nextNode;
		else if (path[0])
			return B_NOT_ALLOWED;

		// are we done?
		if (path[0] == '\0') {
			*_node = nextNode;
			return B_OK;
		}
	}

	return B_ENTRY_NOT_FOUND;
}

```
This function checks if the directory exits and if exits.


+ static status_t
	+ get_node_for_path(Directory *directory, char *path, Node **_node)
	+ {// path = system/mak/neural
		+ directory->Acquire();
		+ // balance Acquire()/Release() calls

		+ while (true) {
			+ Node *nextNode;
			+ char *nextPath;

			+ // walk to find the next path component ("path" will point to a single
			+ // path component), and filter out multiple slashes
			+ for (nextPath = path + 1; nextPath[0] != '\0' && nextPath[0] != '/'; nextPath++);
				// at this point nextPath = /mak/neural
				
			// this condition truncate the '/' from the nextPath.
			+ if (*nextPath == '/') {
				+ *nextPath = '\0';
				+ do
					+ nextPath++;
				+ while (*nextPath == '/');
			+ }
			// now, nextPath = mak/neural/

			// here path is = system
			+ nextNode = directory->[Lookup](#Lookup)(path, true);
			+ directory->Release();

			+ if (nextNode == NULL)
				+ return B_ENTRY_NOT_FOUND;

		path = nextPath;
		if (S_ISDIR(nextNode->Type()))
			directory = (Directory *)nextNode;
		else if (path[0])
			return B_NOT_ALLOWED;

		// are we done?
		if (path[0] == '\0') {
			*_node = nextNode;
			return B_OK;
		}
	}

	return B_ENTRY_NOT_FOUND;
}


## get_node_from

> arg1 = int fd

```

/** Since we don't have directory functions yet, this
 *	function is needed to get the contents of a directory.
 *	It should be removed once readdir() & co. are in place.
 */

Node *
get_node_from(int fd)
{
	Descriptor *descriptor = get_descriptor(fd);
	if (descriptor == NULL)
		return NULL;

	return descriptor->GetNode();
}
```
