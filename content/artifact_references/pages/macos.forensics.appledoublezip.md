---
title: MacOS.Forensics.AppleDoubleZip
hidden: true
tags: [Client Artifact]
---

Search for zip files containing leaked download URLs included by
MacOS users.

MacOS filesystem can represent extended attributes. Similarly to
Windows's ZoneIdentifier, when a file is downloaded on MacOS it also
receives an extended attribute recording where the file was
downloaded from. (See the `Windows.Analysis.EvidenceOfDownload`
artifact)

What makes MacOS different however, is that when a user adds a file
to a Zip file (in Finder, right click the file and select
"compress"), MacOS will also record the extended attributes in the
zip file under the __MACOSX folder.

This is a huge privacy leak because people often do not realize that
the source of downloads for a file is being included inside the zip
file, which they end up sending to other people!

Therefore this artifact can also work on other platforms because Zip
files created by MacOS users can end up on other systems, and
contain sensitive URLs embedded within them.


```yaml
name: MacOS.Forensics.AppleDoubleZip
description: |
  Search for zip files containing leaked download URLs included by
  MacOS users.

  MacOS filesystem can represent extended attributes. Similarly to
  Windows's ZoneIdentifier, when a file is downloaded on MacOS it also
  receives an extended attribute recording where the file was
  downloaded from. (See the `Windows.Analysis.EvidenceOfDownload`
  artifact)

  What makes MacOS different however, is that when a user adds a file
  to a Zip file (in Finder, right click the file and select
  "compress"), MacOS will also record the extended attributes in the
  zip file under the __MACOSX folder.

  This is a huge privacy leak because people often do not realize that
  the source of downloads for a file is being included inside the zip
  file, which they end up sending to other people!

  Therefore this artifact can also work on other platforms because Zip
  files created by MacOS users can end up on other systems, and
  contain sensitive URLs embedded within them.

reference:
  - https://opensource.apple.com/source/Libc/Libc-391/darwin/copyfile.c
  - https://datatracker.ietf.org/doc/html/rfc1740

parameters:
  - name: ZipGlob
    description: Where to search for zip files.
    default: /Users/*/Downloads/*.zip

export: |
    -- Offsets are aligned to 4 bytes
    LET Align(value) = value + value - int(int=value / 4) * 4

    LET Profile = '''[
    ["Header", 0, [
      ["Magic", 0, "uint32b"],
      ["Version", 4, "uint32b"],
      ["Filler", 8, "String", {
          length: 16,
      }],
      ["Count", 24, "uint16b"],
      ["Items", 26, "Array", {
          count: "x=>x.Count",
          type: "Entry",
      }],
      ["attr_header", 84, "attr_header"]
    ]],
    ["Entry", 12, [
      ["ID", 0, "uint32b"],
      ["Offset", 4, "uint32b"],
      ["Length", 8, "uint32b"],
      ["Value", 0, "Profile", {
           type: "ASFinderInfo",
           offset: "x=>x.Offset",
      }]
    ]],
    ["attr_header", 0, [

      # Should be ATTR
      ["Magic", 0, "String", {
          length: 4,
      }],

      ["total_size", 8, "uint32b"],
      ["data_start", 12, "uint32b"],
      ["data_length",16, "uint32b"],
      ["flags", 32, "uint16b"],
      ["num_attr", 34, "uint16b"],
      ["attrs", 36, "Array", {
          count: "x=>x.num_attr",
          type: "attr_t",
      }]
    ]],
    ["attr_t", "x=>Align(value=x.name_length + 11)", [
     ["offset", 0, "uint32b"],
     ["length", 4, "uint32b"],
     ["flags", 8, "uint16b"],
     ["name_length", 10, "uint8"],
     ["name", 11, "String", {
         length: "x=>x.name_length",
     }],
     ["data", 0, "Profile", {
        type: "String",
        type_options: {
            term: "",
            length: "x=>x.length",
        },
        offset: "x=>x.offset",
     }]
    ]]
    ]
    '''

    LET ParseData(data) = if(condition=data =~ "^bplist",
         then=plist(accessor="data", file=data), else=data)

    LET ParseAppleDouble(double_data) = SELECT name AS Key, ParseData(data=data) AS Value
       FROM foreach(row=parse_binary(
            filename=double_data, accessor="data",
            profile=Profile, struct="Header").attr_header.attrs)

sources:
 - query: |
     LET DoubleFiles = SELECT * FROM foreach(row={
        SELECT FullPath AS ZipPath
        FROM glob(globs=ZipGlob)
     }, query={
        SELECT FullPath, pathspec(parse=FullPath) AS PathSpec
        FROM glob(
             globs="__MACOSX/**",
             accessor="zip",
             root=pathspec(DelegatePath=ZipPath))
     })

     SELECT * FROM foreach(row=DoubleFiles,
     query={
       SELECT PathSpec.DelegatePath AS ZipFile,
              PathSpec.Path AS Member,
              Key, Value
       FROM ParseAppleDouble(double_data=read_file(filename=FullPath, accessor="zip"))
     })

```
