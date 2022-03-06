---
title: Understanding gzip - LZ77
date: 2022-03-02T19:34:59+08:00
tags: [gzip, deflate]
categories: 
    - "Base Service"
---

最近工作上需要熟悉gzip软件，直接阅读gzip中的deflate算法那是云里雾里。经过整理网络上的blog，以及阅读[RFC 1951 DEFLATE Compressed Data Format Specification version 1.3](https://datatracker.ietf.org/doc/html/rfc1951)，再度读gzip的deflate代码实现，豁然开朗、受益匪浅，仅以这一系列blog记录下我粗浅的理解。

DEFLATE算法基于LZ77算法和Huffman编码实现流数据压缩。LZ77算法是Abraham Lempel和Jacob Ziv于1977年发布的无损数据压缩算法。LZ77算法是[字典压缩算法](https://en.wikipedia.org/wiki/Dictionary_coder)的一种。字典是encoder维护的包含一组字符串的数据结构，字典压缩过程中，encoder将待压缩的文本在字典中搜索匹配，若找到匹配，则将当前带压缩的字符串用字典中匹配字符串的位置索引来替代，达到缩减数据的效果。LZ77算法的字典是encoder之前已经编码的部分字节序列。

## LZ77算法基本原理
LZ77算法利用文本中字符串具有重复的特点，将后续重复出现的字符串使用`<length, distance>`二元组表示，`length`表示匹配字符串的长度，`distance`表示到之前出现的相同字符串的距离。文本中的重复字符串越多、重复字符串长度越长，LZ77算法则会取得更好的压缩效果。
如下图所示，LZ77算法基于滑动窗口实现。滑动窗口包含两部分：前半部分是已经编码的字符流，称为字典或者search buffer；后半部分是待编码的字符流，称为look-ahead buffer。LZ77算法编码look-ahead buffer中的字符时，会在search buffer中寻找最长匹配字符串，若无匹配则直接输出字符(`literal`)，若存在匹配字符串，则将匹配字符串通过`<length, distance>`编码。DEFLATE算法设定匹配字符串的长度范围是[3, 258]。这个范围是压缩率和算法的时间复杂度之间的平衡。

{% asset_img lz77.png %}

由图可知，LZ77编码的字符流仅包含三种类型的数据：`literal`、`length`、`distance`。实际上，look-ahead buffer长度小于`MIN_LOOKAHEAD`时，滑动窗口便会向前移动以填充后续数据流从而扩大look-ahead buffer。上图所展示，匹配完look-ahead buffer的情况只有在滑动窗口到达字节流末尾，无法向前滑动时出现。

```c
#define WSIZE 0x8000
#define MAX_DIST (WSIZE-MIN_LOOKAHEAD)
#define EOF (-1)

static ulg window_size = (ulg)2*WSIZE;
      unsigned near strstart;      /* start of string to insert */
local unsigned near match_start;   /* start of matching string */
local int           eofile;        /* flag set at end of input file */
local unsigned      lookahead;     /* number of valid bytes ahead in window */
long block_start;
/* window position at the beginning of the current output block. Gets
 * negative when the window is moved backwards.
 */

local void fill_window()
{
    register unsigned n, m;
    //当strstart遍历完滑动窗口时，more = EOF；否则more = 0
    unsigned more = (unsigned)(window_size - (ulg)lookahead - (ulg)strstart);

    if (more == (unsigned)EOF) {
        more--;
    } else if (strstart >= WSIZE+MAX_DIST) {
        // strstart仍在滑动窗口内部，lookahead < MIN_LOOKAHEAD
        // 滑动窗口向前移动WSIZE
        memcpy((char*)window, (char*)window+WSIZE, (unsigned)WSIZE);
        match_start -= WSIZE;
        strstart    -= WSIZE; /* we now have strstart >= MAX_DIST: */
        ...
        block_start -= (long) WSIZE;

        // 更新Search buffer中的hash表
        for (n = 0; n < HASH_SIZE; n++) {
            m = head[n];
            head[n] = (Pos)(m >= WSIZE ? m-WSIZE : NIL);
        }   
        for (n = 0; n < WSIZE; n++) {
            m = prev[n];
            prev[n] = (Pos)(m >= WSIZE ? m-WSIZE : NIL);
        }   
        more += WSIZE;
    }   
    /* At this point, more >= 2 */
    if (!eofile) {
        n = read_buf((char*)window+strstart+lookahead, more);
        if (n == 0 || n == (unsigned)EOF) {
            eofile = 1;
            /* Don't let garbage pollute the dictionary.  */
            memzero (window + strstart + lookahead, MIN_MATCH - 1); 
        } else {
            lookahead += n;
        }   
    }   
}
```
主要解释下几个全局变量：
* `window_size`：滑动窗口大小，默认为64KB
* `lookahead`: look-ahead buffer大小
* `strstart`: 当前待编码的字符在滑动窗口中的位置

当`lookahead < MIN_LOOKAHEAD && !eofile`时会调用`fill_window`更新滑动窗口。会将滑动窗口向前移动`WSIZE`，并更新`match_start`、`strstart`、`block_start`以及search buffer中的Hash表中hash chain上的索引。DEFLATE算法使用hash表优化匹配字符串的查找，hash表的实现会在最后介绍。最后从字符流中读入数据填充到滑动窗口的后半部分：`window+strstart+lookahead`起始位置。

## 惰性匹配（lazy matching）
上图所示的LZ77算法并非最优压缩，当`strstart = 8`，look-ahead buffer中的内容为`abcda`时，即便`abc`在search buffer中能找到匹配字符串，但是紧邻的`bcda`可以在search buffer中找到更长的匹配串。
DEFLATE针对这个情况，设计了惰性匹配（lazy matching）的优化机制：当前匹配的字符串长度为N，会尝试在下一个字符寻找更长的匹配，若在下一个字符能找到更长的匹配（> N），则将当前字符直接输出到编码流中，即`literal`；重复这一过程直至找不到更长的字符串匹配，则将当前的字符找到的匹配以`<length, distance>`二元组输出到编码流中。
```c
local unsigned int near prev_length;
/* Length of the best match at previous step. Matches not greater than this
 * are discarded. This is used in the lazy match evaluation.
 */
local unsigned int max_lazy_match;
/* Attempt to find a better match only when the current match is strictly
 * smaller than this value. This mechanism is used only for compression
 * levels >= 4.
 */
off_t
deflate (int pack_level)
{
...
    /* Process the input block. */
    while (lookahead != 0) {
        // 将字符串window[strstart .. strstart+2]插入到hash表
        INSERT_STRING(strstart, hash_head);

        /* Find the longest match, discarding those <= prev_length.
         */
        prev_length = match_length, prev_match = match_start;
        match_length = MIN_MATCH-1;

        // search buffer中存在匹配字符串；lazy match过程中之前的最长匹配字符串长度 < max_lazy_match；
        // 当前字符与匹配字符串之间的距离 <= MAX_DIST
        if (hash_head != NIL && prev_length < max_lazy_match &&
            strstart - hash_head <= MAX_DIST &&
            strstart <= window_size - MIN_LOOKAHEAD) {
            // 通过hash表，找到当前字符串的最长匹配字符串
            match_length = longest_match (hash_head);
            // 如果匹配的字符串长度超过look-ahead buffer大小，则将匹配字符串的长度设置为lookahead
            if (match_length > lookahead) match_length = lookahead;
            ...
        }

        // lazy match结束，strstart-1存在有效匹配，但是strstart的匹配长度 <= strstart-1的最长匹配长度
        if (prev_length >= MIN_MATCH && match_length <= prev_length) {
            // double check，确认strstart-1的匹配无误
            check_match(strstart-1, prev_match, prev_length);
            // 将strstart-1字符以<length, distance>编码
            flush = ct_tally(strstart-1-prev_match, prev_length - MIN_MATCH);

            // 因为这里用的是前一个字符的匹配，到当前字符匹配时lookahead已经-1，所以此处是-(prev_length-1)
            lookahead -= prev_length-1;
            // strstart-1以及strstart开始的字符串已经插入到hash表中，所以不需要对这两个开始的字符串求hash
            prev_length -= 2;
            // 求匹配字符串中每个字符开始的三字符hash值
            do {
                strstart++;
                INSERT_STRING(strstart, hash_head);
            } while (--prev_length != 0);
            // 不存在前一个字符的匹配，开始lazy match的最开始匹配
            match_available = 0;
            match_length = MIN_MATCH-1;
            strstart++;
            ...
            if (flush) FLUSH_BLOCK(0), block_start = strstart;
        } else if (match_available) {
            // lazy match过程中存在有效的前一个字符匹配的字符串，当前字符串匹配更长；或者当前字符在search buffer中无匹配的字符串
            // 将strstart-1字符以literal编码
            flush = ct_tally (0, window[strstart-1]);
            ...
            if (flush) FLUSH_BLOCK(0), block_start = strstart;
            strstart++;
            lookahead--;
        } else {
            // search buffer中无匹配的字符串
            match_available = 1; 
            strstart++;
            lookahead--;
        }
        // 当look-ahead buffer空间不足时，滑动窗口
        while (lookahead < MIN_LOOKAHEAD && !eofile) fill_window();
    }
    // 如果之前的字符在search buffer中无匹配字符串，将strstart-1字符以literal编码
    if (match_available) ct_tally (0, window[strstart-1]);
                
    return FLUSH_BLOCK(1); /* eof */
} 
```
lazy match的实际实现，我们可以发现：
1. 当前字符与匹配字符串的距离不超过MAX_DIST(32768 - 258 - 3 - 1)，DEFLATE算法规定最大距离为32768
2. 当前字符的最长匹配长度不超过look-ahead buffer长度
3. `match_available = 0`表示strstart匹配不在lazy match的尝试后一个字符匹配字符串比之前匹配字符串长的过程中，可以认为是开始查找最初的匹配字符串。`match_available = 1`表示lazy match过程中存在有效的前一个字符匹配的字符串，或者当前字符在search buffer中无匹配的字符串

## Hash优化
DEFLATE算法通过链式hash表加速search buffer中的重复字符串查找，链式hash表上存储的是长度为3的字符串。
DEFLATE算法的三列函数设计的简单、高效：
```c
local unsigned ins_h;  /* hash index of string to be inserted */

// 初始化hash key
for (j=0; j<MIN_MATCH-1; j++) UPDATE_HASH(ins_h, window[j])

// 更新hash key
/* ===========================================================================
 * Update a hash value with the given input byte
 * IN  assertion: all calls to UPDATE_HASH are made with consecutive
 *    input characters, so that a running hash key can be computed from the
 *    previous key instead of complete recalculation each time.
 */
#define UPDATE_HASH(h,c) (h = (((h)<<H_SHIFT) ^ (c)) & HASH_MASK)

// 将偏移s处的长度为3的字符串插入到hash表中
/* ===========================================================================
 * Insert string s in the dictionary and set match_head to the previous head
 * of the hash chain (the most recent string with same hash key). Return
 * the previous length of the hash chain.
 * IN  assertion: all calls to INSERT_STRING are made with consecutive
 *    input characters and the first MIN_MATCH bytes of s are valid
 *    (except for the last MIN_MATCH-1 bytes of the input file).
 */
#define INSERT_STRING(s, match_head) \
   (UPDATE_HASH(ins_h, window[(s) + MIN_MATCH-1]), \
    prev[(s) & WMASK] = match_head = head[ins_h], \
    head[ins_h] = (s))
```
{% asset_img hash.png %}
对当前字符求hash key时，通过之前的hash key与当前的字符进行异或运算便可在O(1)时间内求得。整个字符流的所有长度为3的字符串的hash key的计算时间复杂度为O(n)。

整个链式hash表的实现也很简单：
1. `head[HASH_SIZE]`数组按照hash key索引，相应位置存储的是最近一次计算得到的hash key的字符位置
2. `prev[WSIZE]`数组存储的是当前字符位置`s`前一个重复字符串的字符位置

因此遍历hash chain可以用如下循环：
```c
INSERT_STRING(strstart, cur_match);
// cur_match <= limit时，匹配字符串超过MAX_DIST
IPos limit = strstart > (IPos)MAX_DIST ? strstart - (IPos)MAX_DIST : NIL;
do {
    ...
    } while ((cur_match = prev[cur_match & WMASK]) > limit);
```