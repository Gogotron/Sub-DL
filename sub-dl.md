#!/usr/local/bin/blaze bash

# Sub-DL

We first use regex to remove the file extension of the movie/episode, using
Perl lookahead.
```sh
without_extension=$(echo "$1" | grep -Po ".+(?=\.)")
```
We then replace the spaces with `+` to make the title url friendly.
```sh
query=$(printf '%s' "$without_extension" | tr ' ' '+' )
```
Then from Open Subtitles we get a page listing the media matching this name. I
don't recall what `sublanguageid-eng` achieves, but I do recall that it fixed
some issue.
```sh
BASE_URL="https://www.opensubtitles.org"
video_results=$(curl -sL "$BASE_URL"/en/search2/sublanguageid-eng/moviename-"$query")
```
With a regex using Perl lookbehind, we get the first number associated to a movie/episode.
```sh
video_id=$(echo "$video_results" | grep -Po "(?<=idmovie-)[0-9]+" | head -n1)
```
From this number we get a page listing different subtitles for the title. I do
recall that `sublanguageid-eng` allows us to ensure only english results in
this case. We sort by number of downloads, descending.
```sh
sub_results=$(curl -s "$BASE_URL"/en/search/sublanguageid-eng/idmovie-"$video_id"/sort-7/asc-0)
```
Again with a regex using Perl lookbehind, we get the first number associated to a subtitle file.
```sh
sub_id=$(echo "$sub_results" | grep -Po "(?<=/en/subtitles/)[0-9]*" | head -n4 | tail -n1)
```
We download the subtitle file into a temporary folder. The file we get is a zip file.
```sh
mkdir -p /tmp/subdl/"$sub_id"
wget --output-document /tmp/subdl/"$sub_id"/"$sub_id".zip "$BASE_URL"/en/subtitleserve/sub/"$sub_id"
```
We list the files in the archive, and with a regex we get the name of the
specific file holding the subtitles (`.srt` extension).
```sh
sub_file_name=$(unzip -l /tmp/subdl/"$sub_id"/"$sub_id".zip | grep -Po " *[0-9]+ +[0-9]{4}-[0-9]{2}-[0-9]{2} +[0-9]{2}:[0-9]{2} +\K(.*\.srt)")
```
We extract that specific file.
```sh
unzip /tmp/subdl/"$sub_id"/"$sub_id".zip "$sub_file_name" -d /tmp/subdl/"$sub_id"
```
And we move it to where the movie/episode is, giving it the same name as the original file so that it may be automatically detected.
```sh
mv /tmp/subdl/"$sub_id"/"$sub_file_name" "$without_extension".srt
```
A potential future step is to use `ffmpeg` to include the subtitles directly in the movie file.
