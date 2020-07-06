# digikam-synology-config
Notes on interoperability between Digikam and Synology Photo Station 6

## Background

### DigiKam

Digikam is an open source photo management tool. Its "killer feature" for my particular application is the ability to store nearly all its data in the photo metadata. It also keeps a database (for speed), but it can be configured to synchronize things like general tags, face tags, ratings, and captions with the EXIF, XMP, and IPTC metadata of the files themselves. It has many wonderful features such as facial recognition.

### Photo Station

Synology Photo Station is a commercial (but free-as-in-beer) photo management tool. It is a PHP-based web gallery that has metadata search, indexing, and basic editing capabilities (including, as with Digikam, tags, face tags, ratings, and captions). 

## Interoperability

The two systems are largeley interoperable. Making no guarantee against race conditions (i.e., use this only for personal photo management), both systems can operate on the same set of photos without misunderstanding one another.

### Filesystem

First of all, we've got to configure the underlying filesystem to work on the same files. I did this by:
- Storing the files primarily on the Synology filesystem
- exporting the "photo" directory via NFS
- adding a "photo" user group with permissions to read and write that filesystem via NFS
- mounting the "photo" NFS directory on the computer running digikam. I did this with autofs; see `synology.autofs` and `auto.misc` for the configuration.
- In digikam, settings -> configure Digikam -> collections -> add the "photo" NFS directory as a "collection on network share", and checking "monitor the albums for external changes". Restart Digikam.

Do not let digikam store its database on the NFS share; that would result in horrible performance! Store the database on a local SSD for best results. This should be the default, if you have initialized digikam on a local photo collection (like /home/user/Pictures) first.

At this point, both pieces of software should be able to "see" the photos.

### Digikam settings

These are available as a digikamrc file in this repo. You do not have to set them via the GUI. However, don't just blow away your digikamrc and replace it with mine; other stuff will break, such as database and file paths. It's a sensibly structured file, modelled after the GUI, and this GUI guide should help you determine which sections you want to copy over.

- Settings -> Configure Digikam -> Metadata 
 - enable all "Write this information to the metadata" settings
 - Settings -> Configure Digikam -> Metadata -> enable all "reading and wrirting metadata" settings EXCEPT for "lazy synchronization"
 - Disable sidecar files
 - Rotation: still deciding what to do here. For now, I edit the files, not the metadata. But I think the metadata will work better when it comes to thumbnail generation.
 - "Advanced" tab: Multiple settings here. Photo Station is not configurable, so we must configure Digikam to read, with priority, from the metadata namespaces written by Photo Station, and to write to them as well.
 
|                |Read (order matters!)                         |Write                         |
|----------------|-------------------------------|-----------------------------|
|Comment	|Exif.Image.ImageDescription<br>Xmp.dc.description<br>Exif.XPComment<br>Xmp.exif.UserComment<br>Xmp.photoshop.headline<br>Xmp.tiff.ImageDescription<br>Xmp.acdsee.notes<br>JPEG/TIFF Comments<br>Iptc.Application2.Caption<br> |Xmp.dc.description<br>Xmp.exif.UserComment<br>Xmp.tiff.ImageDescription<br>Xmp.acdsee.notes<br>JPEG/TIFF Comments<br>Exif.Image.ImageDescription<br>Iptc.Application2.Caption<br>Xmp.photoshop.headline<br>            |
|Rating          |Xmp.xmp.Rating<br>Xmp.acdsee.rating<br>Xmp.MicrosoftPhoto.Rating<br>Exif.Image.0x4746<br>Exif.Image.0x4749<br>Iptc.Application2.Urgency<br>            |Xmp.xmp.Rating<br>Xmp.acdsee.rating<br>Xmp.MicrosoftPhoto.Rating<br>Exif.Image.0x4746<br>Exif.Image.0x4749<br>Iptc.Application2.Urgency<br>Xmp.xmp.RatingPercent<br>(note: set mapping to 0,1,25,50,75,99)<br>            |
|Tags          |Xmp.dc.subject<br>Iptc.Application2.Keywords<br>Exif.Image.XPKeywords<br>Xmp.digiKam.TagsList<br>Xmp.MicrosoftPhoto.LastKeywordXMP<br>Xmp.lr.hierarchicalSubject<br>Xmp.mediapro.CatalogSets<br>Xmp.acdsee.categories<br>|Xmp.dc.subject<br>Iptc.Application2.Keywords<br>Xmp.digiKam.TagsList<br>Xmp.MicrosoftPhoto.LastKeywordXMP<br>Xmp.lr.hierarchicalSubject<br>Xmp.mediapro.CatalogSets<br>Xmp.acdsee.categories<br>Exif.Image.XPKeywords<br>|

### Photo Station Settings

Settings -> Photos ->
- Untested: face recognition might work, since the face tag formats are interoperable natively! I haven't even tried yet, though.

There is nothing else to configure here. I do recommend "always show photo information lightbox", so that the metadata is clearly visible, at least for the admin.

HOWEVER: synology indexing service does not automatically trigger when files are updated, like digikam does. So until I figure out a better way or write a Synology plugin, you will need to run the following command from the synology command line after editing files with Digikam:

Remove the existing thumbnails (they will be regenerated in the next step) (TODO this sucks, can I search for files modified since their eaDir thumbnails were created?)
`find /volume1/photo -iname "@eaDir" -exec rm -rf {} \;`
Reindex the photos:
`synoindex -R photo`
De-index the Digikam trash folder:
`synoindex -D /volume1/photo/.dtrash`

## Conclusion

This isn't complete or perfect. I will continue testing it. Race conditions will always be possible with this configuration; that is, if two users edit the same file at the same time, the system will not prevent or notify their conflicting behavior. 

