---
layout: post
title:  "Getting to the data in Adobe Lightroom–with or without PowerShell"
date:   2012-08-09
categories: 
    - PowerShell
    - Photography
tags: 
    - Adobe Lightroom
    - SQLLite
    - ODBC
    - Lightroom
    - Database
--- 
Some Adobe software infuriates me (Flash), I don’t like their PDF reader and use [Foxit](http://www.foxitsoftware.com/) instead, apps which use Adobe-Air always seem to leak memory. But **I love Lightroom**.  It does things right – like installations – which other Adobe products get wrong. It maintains a “library” of pictures and creates virtual folders of images ( “collections” ) but it maintains metadata in the images files so data stays with pictures when they are copied somewhere else – something some other programs still get badly wrong. My workflow with Lightroom goes something like this.

1.  If I expect to **manipulate the image** at all I set the cameras to **save in RAW**, DNG format not JPG (with my scuba diving camera I use [CHDK](http://chdk.wikia.com/wiki/Downloads) to get the ability to save in DNG)
2.  Shoot pictures – delete any where the camera was pointing at the floor, lens cap was on, studio flash didn’t fire etc. But otherwise **don’t edit in the camera**.
3.  **Copy everything to the computer** – usually I create a folder for a set of pictures and put DNG files into a “RAW” subfolder. I keep full memory cards in filing sleeves meant for 35mm slides..
4.  Using PowerShell I **replace the IMG file-name prefix** with something which tells me what the pictures _are_ but keeps the camera assigned image number.
5.  **Import Pictures into Lightroom** – manipulate them and **export** to the parent folder of the “RAW” one. Make any prints from inside Lightroom. **Delete “dud” images from the Lightroom catalog**.
6.  **Move dud images** out of the RAW folder to their own folder. **Backup everything**. Twice. [I’ve only recently learnt to export the Lightroom catalog information to keep the manipulations with the files]
7.  Remove RAW images from my hard disk

There is one major pain. How do I know which files I have deleted in Lightroom ? I don’t want to delete them from the hard-disk I want to move them later. It turns out Lightroom uses a SQL Lite database and there is a free [Windows ODBC driver for SQL Lite available for download](http://www.ch-werner.de/sqliteodbc/).  With this in place one can create a ODBC data source – point it at a Lightroom catalog and poke about with data. Want a complete listing of your Lightroom data in Excel? ODBC is the answer. But let me issue these warnings:

- Lightroom locks the database files exclusively – you can’t use the ODBC driver and Lightroom at the same time. If something else is holding the files open, Lightroom won’t start.
- The ODBC driver can run UPDATE queries to change the data: do I need to say that is dangerous ? Good.
- There’s no support for this. If it goes wrong, expect Adobe support to say “You did WHAT ?” and start asking about your backups. Don’t come to me either. You can work from a copy of the data if you don’t want to risk having to fall back to one of the backups Lightroom makes automatically

I was interested in 4 sets of data shown in the following diagrams. Below is i**mage information with the Associated metadata**, and file information. Lightroom stores images (Adobe_Images table) IPTC and EXIF metadata link to images – their “image” field joins to the “id_local” primary key in images. Images have a “root file” (in the AgLibraryFile table) which links to a library folder (AgLibraryFolder) which is expressed as a path from a root folder (AgLibraryRootFolder table). The link always goes to the “id_local” field I could get **information about the folders** imported into the catalog just by querying these last two tables (Outlined in red)

![Schema1](/assets/Lightroom-schema-1.png)

The SQL to fetch this data looks like this for just the folders
{% highlight SQL %}
SELECT RootFolder.absolutePath || Folder.pathFromRoot as FullName
FROM   AgLibraryFolder     Folder
JOIN   AgLibraryRootFolder RootFolder ON  RootFolder.id_local = Folder.rootFolder
ORDER BY FullName 
{% endhighlight %}
SQLlite is one of the dialects of SQL which doesn’t accept AS in the FROM part of a SELECT statement . Since I run this in PowerShell I also put a where clause in, which inserts a parameter. To get all the metadata the query looks like this    
{% highlight SQL %}
SELECT    rootFolder.absolutePath || folder.pathFromRoot || rootfile.baseName || '.' || rootfile.extension AS fullName, 
          LensRef.value AS Lens,     image.id_global,       colorLabels,                Camera.Value       AS cameraModel,
          fileFormat,                fileHeight,            fileWidth,                  orientation ,
          captureTime,               dateDay,               dateMonth,                  dateYear,
          hasGPS ,                   gpsLatitude,           gpsLongitude,               flashFired,
          focalLength,               isoSpeedRating ,       caption,                    copyright
FROM      AgLibraryIPTC              IPTC
JOIN      Adobe_images               image      ON      image.id_local = IPTC.image
JOIN      AgLibraryFile              rootFile   ON   rootfile.id_local = image.rootFile
JOIN      AgLibraryFolder            folder     ON     folder.id_local = rootfile.folder
JOIN      AgLibraryRootFolder        rootFolder ON rootFolder.id_local = folder.rootFolder
JOIN      AgharvestedExifMetadata    metadata   ON      image.id_local = metadata.image
LEFT JOIN AgInternedExifLens         LensRef    ON    LensRef.id_Local = metadata.lensRef
LEFT JOIN AgInternedExifCameraModel  Camera     ON     Camera.id_local = metadata.cameraModelRef
ORDER BY FullName
{% endhighlight %}

Note that since some images don’t have a camera or lens logged the joins to those tables needs to be a LEFT join not an inner join. Again the version I use in PowerShell has a Where clause which inserts a parameter.

OK so much for file data – the other data I wanted was about collections. The list of collections is in just one table (AgLibraryCollection) so very easy to query, and but I also wanted to know the images in each collection.

![Schema2](/assets/Lightroom-schema-2.png)

Since one image can be in many collections,and each collection holds many images AgLibraryCollectionImage is a table to provide a many to relationship. Different tables might be attached to AdobeImages depending on what information one wants from about the images in a collection, I’m interested only in mapping files on disk to collections in Lightroom, so I have linked to the file information and I have a query like this.
{% highlight SQL %}
SELECT   Collection.name AS CollectionName ,
         RootFolder.absolutePath || Folder.pathFromRoot || RootFile.baseName || '.' || RootFile.extension AS FullName
FROM     AgLibraryCollection Collection
JOIN     AgLibraryCollectionimage cimage     ON collection.id_local = cimage.Collection
JOIN     Adobe_images             Image      ON      Image.id_local = cimage.image
JOIN     AgLibraryFile            RootFile   ON   Rootfile.id_local = image.rootFile
JOIN     AgLibraryFolder          Folder     ON     folder.id_local = RootFile.folder
JOIN     AgLibraryRootFolder      RootFolder ON RootFolder.id_local = Folder.rootFolder
ORDER BY CollectionName, FullName
{% endhighlight %}
Once I have an ODBC driver (or an OLE DB driver) I have a ready-made PowerShell template for getting data from the data source. So I wrote functions to let me do :    
`Get-LightRoomItem -ListFolders -include $pwd  ` To list folders, below the current one, which are in the LightRoom Library,    
`Get-LightRoomItem  -include "dive"  ` To list files in LightRoom Library where the path contains  "dive" in the folder or filename,    
`Get-LightRoomItem | Group-Object -no -Property "Lens" | sort count | ft -a count,name`    
To produce a summary of lightroom items by lens used. And    
```
$paths = (Get-LightRoomItem -include "$pwd%dng" | select -ExpandProperty path)  ;   dir *.dng |
           where {$paths -notcontains $_.FullName} | move -Destination scrap -whatif
```    
Stores paths of lightroom items in the current folder ending in .DNG in $paths;  then gets files in the current folder and moves those which are not in $paths (i.e. in Lightroom.) specifying  -Whatif allows the files to be confirmed before being moved.

`Get-LightRoomCollection` lists all collections and    
`Get-LightRoomCollectionItem -include musicians | copy -Destination e:\raw\musicians` will copy the original files in the “musicians” collection to another disk

I’ve shared the PowerShell code on Skydrive