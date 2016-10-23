# zip_tricks

[![Build Status](https://travis-ci.org/WeTransfer/zip_tricks.svg?branch=master)](https://travis-ci.org/WeTransfer/zip_tricks)

Allows streaming, non-rewinding ZIP file output from Ruby.
Spiritual successor to [zipline](https://github.com/fringd/zipline)

Requires Ruby 2.1+ syntax support and a working zlib (all available to jRuby as well).

## Create a ZIP file without size estimation, compress on-the-fly)

When you compress on the fly and use data descriptors it is not really possible to compute the file size upfront.
But it is very likely to yield good compression - especially if you send things like CSV files.

```ruby
out = my_tempfile # can also be a socket
ZipTricks::Streamer.open(out) do |zip|
  zip.write_stored_file('mov.mp4.txt') do |sink|
    File.open('mov.mp4', 'rb'){|source| IO.copy_stream(source, sink) }
  end
  zip.write_deflated_file('long-novel.txt') do |sink|
    File.open('novel.txt', 'rb'){|source| IO.copy_stream(source, sink) }
  end
end
```

## Send the same ZIP file from a Rack response

Create a `RackBody` object and give it's constructor a block that adds files.
The block will only be called when actually sending the response to the client
(unless you are using a buffering Rack webserver, such as Webrick).

```ruby
body = ZipTricks::RackBody.new do | zip |
  zip.write_stored_file('mov.mp4') do |sink| # Those MPEG4 files do not compress that well
    File.open('mov.mp4', 'rb'){|source| IO.copy_stream(source, sink) }
  end
  zip.write_deflated_file('long-novel.txt') do |sink|
    File.open('novel.txt', 'rb'){|source| IO.copy_stream(source, sink) }
  end
end
[200, {'Transfer-Encoding' => 'chunked'}, body]
```

## Send a ZIP file of known size, with correct headers

Use the `SizeEstimator` to compute the correct size of the resulting archive.

```ruby
# Prepare the response body. The block will only be called when the response starts to be written.
zip_body = ZipTricks::RackBody.new do | zip |
  zip.add_stored_entry(filename: "myfile1.bin", size: 9090821, crc32: 12485)
  zip << read_file('myfile1.bin')
  zip.add_stored_entry(filename: "myfile2.bin", size: 458678, crc32: 89568)
  zip << read_file('myfile2.bin')
end

# Precompute the Content-Length ahead of time
bytesize = ZipTricks::SizeEstimator.estimate do |z|
 z.add_stored_entry(filename: 'myfile1.bin', size: 9090821)
 z.add_stored_entry(filename: 'myfile2.bin', size: 458678)
end

[200, {'Content-Length' => bytesize.to_s}, zip_body]
```

## Other usage examples

Check out the `examples/` directory at the root of the project. This will give you a good idea
of various use cases the library supports.

## Writing ZIP files using the Streamer bypass

You do not have to "feed" all the contents of the files you put in the archive through the Streamer object.
If the write destination for your use case is a `Socket` (say, you are writing using Rack hijack) and you know
the metadata of the file upfront (the CRC32 of the uncompressed file and the sizes), you can write directly
to that socket using some accelerated writing technique, and only use the Streamer to write out the ZIP metadata.

```ruby
# io has to be an object that supports #<<
ZipTricks::Streamer.open(io) do | zip |
  # raw_file is written "as is" (STORED mode).
  # Write the local file header first..
  zip.add_stored_entry(filename: "first-file.bin", size: raw_file.size, crc32: raw_file_crc32)
  
  # then send the actual file contents bypassing the Streamer interface
  io.sendfile(my_temp_file)
  
  # ...and then adjust the ZIP offsets within the Streamer
  zip.simulate_write(my_temp_file.size)
end
```

## Computing the CRC32 value of a large file

`BlockCRC32` computes the CRC32 checksum of an IO in a streaming fashion.
It is slightly more convenient for the purpose than using the raw Zlib library functions.

```ruby
crc = ZipTricks::StreamCRC32.new
crc << large_file.read(1024 * 12) until large_file.eof?
...

crc.to_i # Returns the actual CRC32 value computed so far
...
# Append a known CRC32 value that has been computed previosuly
crc.append(precomputed_crc32, size_of_the_blob_computed_from)
```

## Contributing to zip_tricks
 
* Check out the latest master to make sure the feature hasn't been implemented or the bug hasn't been fixed yet.
* Check out the issue tracker to make sure someone already hasn't requested it and/or contributed it.
* Fork the project.
* Start a feature/bugfix branch.
* Commit and push until you are happy with your contribution.
* Make sure to add tests for it. This is important so I don't break it in a future version unintentionally.
* Please try not to mess with the Rakefile, version, or history. If you want to have your own version, or is otherwise necessary, that is fine, but please isolate to its own commit so I can cherry-pick around it.

## Copyright

Copyright (c) 2016 WeTransfer. See LICENSE.txt for further details.
