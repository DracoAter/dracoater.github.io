---
layout: post
title: Extracting Files From tar.gz With Ruby
date: '2013-10-02T18:26:00.000+03:00'
author: Juri Timošin
tags:
- tar
- gzip
- Ruby
modified_time: '2013-10-03T14:13:46.682+03:00'
blogger_id: tag:blogger.com,1999:blog-360329120074358364.post-6121417887019689343
blogger_orig_url: http://dracoater.blogspot.com/2013/10/extracting-files-from-targz-with-ruby.html
---

[1]: http://stackoverflow.com/search?q=ruby+tar+gzip
[2]: http://en.wikipedia.org/wiki/Tar_%28computing%29
[3]: http://stackoverflow.com/q/2078778/170230

I always thought that it should be a trivial task. There are even some
[stackoverflow][1] answers on that topic, but there is actually a catch that
none of the answers talks about.

<!--more-->

Originally [tar did not support paths longer than 100 chars][2]. GNU tar is better and they
implemented support for longer paths, but it was made through a *hack* called
`././@LongLink` (see [here][3]). Shortly speaking, if you stumble upon an entry in
tar archive which path equals to above mentioned `././@LongLink`, that means that the
following entry path is longer than 100 chars and is truncated. The full path of the following
entry is actually the value of the current entry. So when extracting files from tar we also must
have in mind this possibility.

{% highlight ruby linenos %}
require 'rubygems/package'
require 'zlib'

TAR_LONGLINK = '././@LongLink'
tar_gz_archive = '/path/to/archive.tar.gz'
destination = '/where/extract/to'

Gem::Package::TarReader.new( Zlib::GzipReader.open tar_gz_archive ) do |tar|
  dest = nil
  tar.each do |entry|
    if entry.full_name == TAR_LONGLINK
      dest = File.join destination, entry.read.strip
      next
    end
    dest ||= File.join destination, entry.full_name
    if entry.directory?
      FileUtils.rm_rf dest unless File.directory? dest
      FileUtils.mkdir_p dest, :mode => entry.header.mode, :verbose => false
    elsif entry.file?
      FileUtils.rm_rf dest unless File.file? dest
      File.open dest, "wb" do |f|
        f.print entry.read
      end
      FileUtils.chmod entry.header.mode, dest, :verbose => false
    elsif entry.header.typeflag == '2' #Symlink!
      File.symlink entry.header.linkname, dest
    end
    dest = nil
  end
end
{% endhighlight ruby %}
