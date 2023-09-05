---
title: "Backend Python:Temporary Files in Python"
date: 2019-09-05T11:01:54+03:00
draft: false 
categories: ["python"]
tags: ["python"]
---
Let me start with an actual use case scenario. As a backend developer, I need to process the user uploaded file data all the time. Here temporary files shine. The best part about these is that they make cleanup easier. If you make a real file, you need to use some OS level utility to manually delete the file after you are done with it. If you are using temporary files – option-1, then just by closing the file, you are guaranteed that OS will later clear it up. If using option-2, it will get garbage collected.

there are 2 options:
## option 1:
make an actual temporary file. As in, a file that exists on the disk, just in the temporary file area.

```python
import tempfile
import os
        
f = tempfile.NamedTemporaryFile(delete=False)
try:
    # do whatever. f is exactly like a file
    # it actually exists on the hard disk in the temporary files area
            
finally:
    f.close()
    os.unlink(f.name)
```
You can also just create `tempfile.TemporaryFile()` instead of `NamedTemporaryFile`. In that case, in the finally block, you won’t need `os.unlink(f.name)` since this one will have no name.

## option 2:
Create an in-memory file like object. This is technically not a file, just a buffer, but it supports many file-like operations like `f.read()`, `f.write()`, `f.seek()`.
What you need is to identify what kind of file you need and use either ByteIO (for file like object containing bytes, like image blob) or StringIO (for storing text).

Here I am creating an in-memory CSV file.
```python
base_url = 'http://example.com'
data = [
    ['sku', 'url']
    , [1, '{}/path/to'.format(base_url)]
    , [2, '{}/path/to2'.format(base_url)]
]
f = StringIO()
w = csv.writer(f, delimiter='\t')
for row in data:
    w.writerow(row)
f.seek(0)
```
Now the object `f` quacks like a file. If you are writing an endpoint test that uploads a file, you can create a test file like so and upload.

Here there is no need to close the file as the buffer is automatically garbage collected.

```python
def test_upload_video(self):
    base_url = 'http://example.com'
    data = [
        ['sku', 'url']
        , [1, '{}/path/to'.format(base_url)]
        , [2, '{}/path/to2'.format(base_url)]
    ]
    f = StringIO()
    w = csv.writer(f, delimiter='\t')
    for row in data:
        w.writerow(row)
    f.seek(0)

    self.client.post('/api/v1/some_endpoint/', {'uploaded_file': f})
```
And then, in the view, you will be able to access the csv file. Note, here `uploaded_file` is not a keyword, just a dictionary key.