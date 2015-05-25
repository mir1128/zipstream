Introduction
This article presents two zipped STL iostream implementation based on the library zlib (see download link above) and bzip2 (see download link above). This means that you can easily manipulate zipped streams like any other STL ostream/istream.

To give you an idea, consider following snippet that prints "Hello World":

Hide   Copy Code
ostringstream output_buffer;
// writing data
output_buffer<<"Hello world"<<endl 
Now, the same snippet but with zipped output using zlib:

Hide   Copy Code
// zip_ostream uses output_buffer as output buffer :)
zip_ostream zipper( output_buffer );

// writing data as usual
zipper<<"Hello world"<<endl 
or using bzip2:

Hide   Copy Code
// zip_ostream uses output_buffer as output buffer :)
bzip2_ostream bzipper( output_buffer );

// writing data as usual
bzipper<<"Hello world"<<endl 
As you can see adding zipped buffers into your existing applications is quite straightforward. To summarize, let's see some quick facts about zipstream and bzip2stream:

STL compliant,
any-stream-to-any-stream support,
char, wchar_t support,
fining tuning of compression properties,
support custom allocators (New!)
Why another wrapper? Why not use gzstream?
Writing wrappers around the zlib library is popular on CodeProject. If you search for 'zip' you will find at least 14 articles on the topic. Moreover, if you crawl on the web and especially on zlib home page, you can find dozens of other wrappers.

So why another wrapper? Well, none of the wrappers are fully STL compliant. Ok, this is not true since gzstream (see download link above) implements fstream-like STL streams. However, gzstream has three drawbacks:

It does not allow buffer to buffer compression since it is based on gzip i/o methods: only file to buffer or buffer to file are supported,
it is licensed under LGPL which makes it difficult to use in commercial apps,
it does not support wchar_t
Last reason to write this wrapper: it is a good exercise to understand and implement iostreams.

Wrapper architecture
The three drawbacks of gzstream pushed me to re-implement an STL wrapper for zlib (and later on, do some cut and paste to get bzip2 working).

This wrapper takes a user defined i/ostream to write or read compressed data. This approach is quite flexible since the user can give any stream (istringstream, ifstream, or custom stream) to store or load the compressed data.

Internally zip_stream acts as a triple buffer: the streambuf object in itself, zlib library and the user-defined stream. For example, during the compression process, the buffers are used:

first buffer: the data to compress is buffered into a streambuf object,
second buffer: when overflow is called, the first buffer data is sent to zlib which also buffers the data internally. If zlib outputs data, it is sent to the user-defined stream,
third buffer: the user-defined stream is buffered.
Some care must be taken when flushing: you must use the method zflush that will first flush the streambuf, then flush the zlib buffer, then flush the user-defined stream. Note that you should avoid flushing as it degrades compression.

Implementing iostreams
Since I'm not an STL expert, I will very briefly discuss this part. There's room for a tutorial on this topic...

To implement custom iostream, you need to take the following steps:

implement a custom my_streambuf, inherited from streambuf. You need to override the virtual methods sync, underflow and overflow. sync and overflow are used in output streams and underflow is used in input streams.
implement a custom my_ostream, inherited from ostream. It will use my_streambuf
implement a custom my_istream, inherited from istream. It will use my_streambuf as stream buffer.
Class quick reference
All the zlib classes are in the zlib_stream namespace and all the bzip2 classes are in the bzip2_stream namespace.

The two main classes of the zlib wrapper are basic_zip_ostream and basic_zip_istream which implement respectively compression and decompression and behave like classic basic_ostream and basic_istream.

Classical typedef are also provided for these classes:

zip_ostream, zip_istream for char streams
wzip_ostream, wzip_istream for wchar_t streams.
The bzip2 classes have similar names, just replace zlib by bzip2: basic_zip_streambuf becomes basic_bzip2_streambuf.

basic_zip_ostream

This class inherits from basic_ostream:

Hide   Copy Code
template<
    typename Elem, 
    typename Tr = char_traits<Elem;>,
    typename ElemA = std::allocator<Elem>,
    typename ByteT = unsigned char,
    typename ByteAllocatorT = std::allocator<ByteT> 
    >
basic_zip_ostream : public basic_ostream<Elem, Tr>
where

Elem,Tr are the classical basic_ostream template parameters,
ElemA is the allocator for a Elem buffer used internally,
ByteT is the byte type used internally (you should not change that),
ByteAT is a custom allocator for a ByteT buffer used internally
Constructor

Hide   Copy Code
basic_zip_ostream( 
    ostream_reference ostream_, 
    bool is_gzip_ = false,
    size_t level_ = Z_DEFAULT_COMPRESSION,
    EStrategy strategy_ = DefaultStrategy,
    size_t window_size_ = 15,
    size_t memory_level_ = 8,
    size_t buffer_size_ = 4096
);
ostream_ is a user defined output stream,
is_gzip_, true if you want to add the gzip header and footer,
level_, compression level 0, bad and faster to 9 max and slower,
strategy_, compression strategy, see EStrategy enum,
window_size_, memory_level_ are advanced zlib settings, check zlib manual,
buffer_size_, read buffer size
Note that if you choose the gzip option, a header will be automatically added in the constructor and the gzip footer (CRC + data size) will be added in the destructor.

Other methods

Flush all buffers (zlib and ostream):
Hide   Copy Code
basic_zip_ostream& zflush()
This method must be called before using the compressed data! Since zlib does it's own buffering and ostream::flush is not virtual there is no way to avoid this problem.

Return the CRC of the uncompressed data:
Hide   Copy Code
long get_crc();
Return the uncompressed data size:
Hide   Copy Code
long get_in_size();
Return the compressed data size:
Hide   Copy Code
long get_out_size();
Predefined typedefs

Hide   Copy Code
typedef basic_zip_ostream<char> zip_ostream;
typedef basic_zip_ostream<wchar_t> zip_wostream;
basic_zip_istream
This class inherits from basic_istream:

Hide   Copy Code
template<
    typename Elem, 
    typename Tr = char_traits<Elem;>,
    typename ElemA = std::allocator<Elem>,
    typename ByteT = unsigned char,
    typename ByteAT = std::allocator<ByteT> 
    >
basic_zip_istream : public basic_istream<Elem, Tr>
Constructor

Hide   Copy Code
basic_zip_istream( 
    istream_reference istream_, 
    size_t window_size_ = 15,
    size_t read_buffer_size_ = 1024 * 10,
    size_t input_buffer_size_ = 1024 * 5
)
istream_, input stream containing the compressed data,
window_size_, should be compatible with compression window size,
read_buffer_size_, size of the streambuf buffer size,
input_buffer_size_, size of the zlib input buffer size
Other methods

Tells if it is a gzip file:
Hide   Copy Code
bool is_gzip() const
Checks CRC (must be a gzip file)
Hide   Copy Code
bool check_crc() const
Return the CRC of the uncompressed data:
Hide   Copy Code
long get_crc() const;
Return the uncompressed data size:
Hide   Copy Code
long get_out_size() const;
Return the compressed data size:
Hide   Copy Code
long get_in_size() const;
Predefined typedefs

Hide   Copy Code
typedef basic_zip_istream<char> zip_istream;
typedef basic_zip_istream<wchar_t> zip_wistream;
How to ...
All the following examples are valid for both zlib and bzip2 wrappers.

Compress to a buffer

Hide   Copy Code
ostringstream buffer;
zip_ostream zipper(buffer);

// writing stuff
zipper<<...

//flushing VERY IMPORTANT!
zipper.zflush();

// buffer.str() is ready
Compress to a file

Hide   Copy Code
ofstream file("test.zip",ios::out | ios::binary);
{
zip_ostream zipper(file, true /* gzip file*/);

// writing stuff
zipper<<...

} // the stream is flushed, the destructor is called and gzip header appended
// the file is ready
Decompress from a buffer

Hide   Copy Code
istringstream buffer;
zip_istream unzipper(buffer);

// reading stuff
unzipper>>...
Decompress from a file

Hide   Copy Code
ifstream file("test.zip", ios::in | ios::binary);
zip_istream unzipper(file);

// reading stuff
unzipper>>...

// if the file was gzip, we can check the crc
if (unzipper.is_gzip())
    std::cout<<"crc check: "<<( unzipper.check_crc() ? "ok" : "failed");
Using it in your project
Zlib wrapper

Read the license terms,
Copy zipstream.hpp and zipstream.ipp in your include directory,
Make sure zlib is available,
Add #include "zlibstream.hpp" to include the headers,
bzip2 wrapper

Read the license terms,
Copy bzip2stream.hpp and bzip2stream.ipp in your include directory,
Make sure zlib is available,
Add #include "bzip2stream.hpp" to include the headers,
History
30-09-2003, 1.7, added custom allocators (suggestion of <forgot, send me your e-mail>), fixed warning in ostream constructors
21-09-2003, fixed bugs with CRC and size, writing thanks to Jeroen Dirks and gigimenegolo
08-08-2003, 1.5
Fixed gzip footer problem: CRC is read
data is put back in the buffer when zip file has finished
07-18-2003, 1.4 Fixed gzip header problem.
07-3-2003, 1.3, add bzip2 wrapper
07-2-2003, 1.2, wchar_t working
07-2-2003, 1.1, fixed bugs in gzip header and zip_to_stream
07-01-2003, 1.0, initial release
Reference
[1] Zlib home page,
[2] gzstream home page
License
These zlib and bzip2 wrappers are licensed under the zlib/libpng license.


License
This article has no explicit license attached to it but may contain usage terms in the article text or the download files themselves. If in doubt please contact the author via the discussion board below.

A list of licenses authors might use can be found here
