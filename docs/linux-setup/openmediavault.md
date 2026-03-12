# Openmediavault

For most of the services we're going to setup, we want to have a dedicated or shared "data" folder that they use. For example, where all the nextcloud data goes, where your gitea remote repositories live, etc. There are a few ways to enable that, but here we're going to first make a NAS VM and use "shared folders" and nfs connections to our other services. This way, our storage is more or less managed in one central location.

## Shared folders

A shared folder is simply a folder entity that can be used in other parts of OMV, e.g. through samba share or nfs shares. You can specify both the name and path of these shared files, and you can make use of the path to organise the shared folders in the underlying folder structure.

For example, if you want to have all your service (e.g. nextcloud, jellyfin) folders under the same parent `/SERVICES/JELLYFIN` folder, you can specify that in the `Relative path` field.

To make a new shared folder, navigate to `Storage > Shared Folders` and click the add button.

| Field         | Info                                                                                                                                                                                                                                                 |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Name          | This is the name that identifies the shared folder throughout OMV. This is also the name that you use to connect to the folder using nfs. E.g. if you name the folder `EXAMPLE-SHARE` then you would connect to it using `10.10.1.XX:/EXAMPLE-SHARE` |
| File system   | Which logical drive this folder should be.                                                                                                                                                                                                           |
| Relative path | The path of the folder. You can use this to organise the structure of your shared folders.                                                                                                                                                           |
| Permissions   | Who can read/write to this shared folder.                                                                                                                                                                                                            |

## Shared folders with nfs

Navigate to `Services > NFS > Shares`. Click the add button, then specify the client IP address, and whether the client should have `read` or `read/write` access.

Make sure to save and update after creating this share.

## Shared folders with multiple clients using nfs

To share a shared folder with multiple clients, simply make multiple entries in `Services > NFS > Shares`. These should then appear in your `/etc/exports` on the same line, e.g.

```bash
$ cat /etc/exports

/export/EXAMPLE-EXPORT 10.10.1.XX(xx,xx,xx,xx) 10.10.1.YY(yy, yy, yy, yy)
```

I would recommend checking this the first time you do this to make sure it got added correctly.
