# Adaptive Multimedia Streaming Framework for ndnSIM  - Tutorial
**Note:** This tutorial requires you to have a working installation of [AMuSt-ndnSIM](http://github.com/ChristianKreuzberger/AMuSt-ndnSIM). See the information about installation [here](http://github.com/ChristianKreuzberger/AMuSt-ndnSIM#installing-procedure).

Within this tutorial, we will try to cover simple examples like file-transfers, up to building a large example with several multimedia streaming clients.

AMuSt-ndnSIM consists of two major components: a generic file transfer for NDN and a multimedia streaming specific app. Therefore, this tutorial is organized as follows: Section 1 explains how the file transfers are organized and implemented. Section 2 explains how adaptive multimedia streaming can be used in AMuSt-ndnSIM. 
Last but not least, we will show how to generate a large network using BRITE in Section 3.

Throughout the tutorial we will assume that you are placing any code in the ndnSIM/ns-3/scratch folder, and that you are executing it from the ndnSIM/ns-3 folder.


**Table of Contents**  *generated with [DocToc](http://doctoc.herokuapp.com/)*

- [Adaptive Multimedia Streaming Framework for ndnSIM  - Tutorial](#)
	- [1. File Transfers](#)
		- [Basics](#)
		- [Hosting Content (Producer)](#)
		- [Requesting Content (Consumer)](#)
		- [Full Example: Basic File Transfer](#)
		- [Tracing File Transfers](#)
		- [Pipe-Lining Interests (Enhanced File Consumer)](#)
		- [Testing File Transfers with Real Data](#)
		- [Tracing Several File Consumers](#)
		- [Hosting Virtual Files](#)
		- [Summary](#)
	- [2. Multimedia Streaming](#)
		- [Hosting Multimedia/DASH Content](#)
		- [Hosting Actual DASH Content](#)
			- [DASH/AVC Streaming](#)
			- [DASH/SVC Streaming](#)
			- [Creating a Server](#)
		- [Hosting Real Content using FakeFileServer](#)
		- [Hosting Virtual Content using FakeMultimediaServer](#)
	- [Basics: Using the Multimedia Consumer](#)
		- [AVC Content](#)
		- [SVC Content](#)
	- [Multimedia Consumers Options](#)
	- [Multimedia Consumers and Tracers](#)
		- [Other Tracers](#)
	- [3. Building Large Networks with BRITE and Installing Multimedia Clients](#)
	- [Basics](#)
	- [Random Network](#)
	- [Installing the NDN Stack, Multimedia Clients, Routing, ...](#)




------------------
## 1. File Transfers
Unfortunately, at the time of writing this extension to the simulator, ndnSIM did not provide any implementation for transfering files. In addition, fragmentation of packets was also not implemented, leading to a packet size of roughly 1400 bytes, similar of TCP transfer (just without fragmentation). The only implemented application that come close are the consumer and producer app (resp. ndn-app-consumer-cbr and ndn-app-producer). 

First, we will talk about the basics of the file transfer, followed by an example of how to host and request the content. Last but not least, we will provide a full scenario and information about tracing.

### Basics
Therefore we implemented a very basic FileServer (resp. ``ns3::ndn::FileServer``) and a FileConsumer (resp. ``ns3::ndn::FileConsumer``) for basic file transfers. Configuring the FileServer is simple: It only needs to know under which prefix (e.g., /myprefix) it needs to host files and where to find files to host. The FileServer automatically handles fragmentation/segmentation of files that do not fit within a single packet. Based on the Maximum Transmission Unit (MTU), the FileServer aims to always fully utilize the MTU by automatically maximizing the payload size. This immediately leads to a limitation of the network: The MTU needs to be the same for the whole network (e.g., 1500).

We achieved fragmentation by providing a so called Manifest for each file, which contains the number of fragments to request and the file size. A typical file transfer based on this implementation looks like this: first, an interest for ``/prefix/app/File1.txt/Manifest`` is issued. Second, the consumer will start to sequentially issue Interests for the chunks: ``/prefix/app/File1.txt/1``, ``/prefix/app/File1.txt/2``, ... ``/prefix/app/File1.txt/#n`` (where #n is the number of fragments). The numbers (also referred to as sequence numbers) are appeneded to the interest with a binary encoding by using ``ndn::Name::appendSequenceNumber``. 
The FileServer will respond with the respective payload, and the file consumer will stop requesting once it has received the whole file. The consumer also handles timeouts and re-transmissions if necessary. 


### Hosting Content (Producer)
Before we start, we would like to point out that our examples are very similar to the basic examples provided by the [ndnSIM tutorial page](http://ndnsim.net/2.1/examples.html). Please have a look at the examples there before you start working with AMuSt-ndnSIM.

As already mentioned, hosting content does not require any special attention. You can host content on any ndnSIM node, similar as you would do with the ``ns3::ndn::Producer``, by using the ``ns3::ndn::FileServer`` application, as shown in the following sourcecode:
```cplusplus
  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /prefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("ContentDirectory", StringValue("/home/username/somedata/"));
  producerHelper.Install(nodes.Get(2)); // install to a node from the nodecontainer
```
This will make the directory  ``/home/someuser/somedata/`` and all contents (including sub-directories) fully available under the ndn prefix ``/myprefix`` within the simulator. 
You can install this producer on as many nodes you want. Just make sure to also provide routing information, the same way as you would for a "plain" ndn-producer.

```cplusplus
  // Optional: Routing  (if and where applicable)
  ndn::GlobalRoutingHelper ndnGlobalRoutingHelper;
  ndnGlobalRoutingHelper.InstallAll();
  ...
  ndnGlobalRoutingHelper.AddOrigins("/myprefix", nodes.Get(2));
  ...
  ndn::GlobalRoutingHelper::CalculateRoutes();
```

### Requesting Content (Consumer)
Similar to the example above, the client, also called ``FileConsumer``, needs to be installed on some node:
```cplusplus
  // Consumer
  ndn::AppHelper consumerHelper("ns3::ndn::FileConsumer");
  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file1.img"));
  
  consumerHelper.Install(nodes.Get(0)); // install to some node from nodelist
```
This simple piece of code will automatically request file1.txt, similar to what ``ns3::ndn::Consumer`` would do.

### Full Example: Basic File Transfer
First, create a data directory in your home directory, and create a file of 10 Megabyte size:
```bash
mkdir ~/somedata
fallocate -l 10M ~/somedata/file1.img
```

Then, create a .cpp file with the following content in your scenario subfolder (see [AMuSt-ndnSIM/examples/ndn-file-simple-example1.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/blob/master/examples/ndn-file-simple-example1.cpp)):
```cplusplus
  // setting default parameters for PointToPoint links and channels
  Config::SetDefault("ns3::PointToPointNetDevice::DataRate", StringValue("10Mbps"));
  Config::SetDefault("ns3::PointToPointChannel::Delay", StringValue("10ms"));
  Config::SetDefault("ns3::DropTailQueue::MaxPackets", StringValue("20"));

  // Read optional command-line parameters (e.g., enable visualizer with ./waf --run=<> --visualize
  CommandLine cmd;
  cmd.Parse(argc, argv);

  // Creating nodes
  NodeContainer nodes;
  nodes.Create(3); // 3 nodes, connected: 0 <---> 1 <---> 2

  // Connecting nodes using two links
  PointToPointHelper p2p;
  p2p.Install(nodes.Get(0), nodes.Get(1));
  p2p.Install(nodes.Get(1), nodes.Get(2));

  // Install NDN stack on all nodes
  ndn::StackHelper ndnHelper;
  ndnHelper.SetDefaultRoutes(true);
  ndnHelper.InstallAll();

  // Choosing forwarding strategy
  ndn::StrategyChoiceHelper::InstallAll("/myprefix", "/localhost/nfd/strategy/best-route");

  // Consumer
  ndn::AppHelper consumerHelper("ns3::ndn::FileConsumer");
  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file1.img"));

  consumerHelper.Install(nodes.Get(0)); // install to some node from nodelist

  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /prefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("ContentDirectory", StringValue("/home/someuser/somedata/"));
  producerHelper.Install(nodes.Get(2)); // install to some node from nodelist

  ndn::GlobalRoutingHelper ndnGlobalRoutingHelper;
  ndnGlobalRoutingHelper.InstallAll();

  ndnGlobalRoutingHelper.AddOrigins("/myprefix", nodes.Get(2));
  ndn::GlobalRoutingHelper::CalculateRoutes();

  Simulator::Stop(Seconds(600.0));

  Simulator::Run();
  Simulator::Destroy();

  NS_LOG_UNCOND("Simulation Finished.");
``` 

You can now compile and start the scenario using
```bash
./waf --run ndn-file-simple-example1
```
Though you will not see any output. We will talk about generating some output based on traces from file transfers in the next section. For now, if you just want to verify that your file transfer is working, use the visualizer:
```bash
./waf --run ndn-file-simple-example1 --vis
```

### Tracing File Transfers
For more information about the file transfer, especially the download speed, we have implemented file transfer tracers on the consumer side, similar to existing tracers in ndnSIM. At first, we need to add 3 functions to  our previous scenario. They will deal with printing information to the console:
```cplusplus
// FileDownloadedTrace is called when the file download finished
void
FileDownloadedTrace(Ptr<ns3::ndn::App> app, shared_ptr<const ndn::Name> interestName, double downloadSpeed, long milliSeconds)
{
  std::cout << "Trace: File finished downloading: " << Simulator::Now().GetMilliSeconds () << " "<< *interestName <<
     " Download Speed: " << downloadSpeed/1000.0 << " Kilobit/s in " << milliSeconds << " ms" << std::endl;
}

// FileDownloadedManifestTrace is called when the Manifest has been received, along with the file size in the manifest
void
FileDownloadedManifestTrace(Ptr<ns3::ndn::App> app, shared_ptr<const ndn::Name> interestName, long fileSize)
{
  std::cout << "Trace: Manifest received: " << Simulator::Now().GetMilliSeconds () <<" "<< *interestName << " File Size: " << fileSize << std::endl;
}

// FileDownloadStartedTrace is called when the file download has started (before the manifest has been received)
void
FileDownloadStartedTrace(Ptr<ns3::ndn::App> app, shared_ptr<const ndn::Name> interestName)
{
  std::cout << "Trace: File started downloading: " << Simulator::Now().GetMilliSeconds () <<" "<< *interestName << std::endl;
}
```

Next, we need to make sure to connect them as trace sources:
```cplusplus
  // Connect Tracers
  Config::ConnectWithoutContext("/NodeList/*/ApplicationList/*/FileDownloadFinished",
                               MakeCallback(&FileDownloadedTrace));
  Config::ConnectWithoutContext("/NodeList/*/ApplicationList/*/ManifestReceived",
                               MakeCallback(&FileDownloadedManifestTrace));
  Config::ConnectWithoutContext("/NodeList/*/ApplicationList/*/FileDownloadStarted",
                               MakeCallback(&FileDownloadStartedTrace));
```

You can find the whole sourcecode under [AMuSt-ndnSIM/examples/ndn-file-simple-example2-tracers.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/blob/master/examples/ndn-file-simple-example2-tracers.cpp). The output of this looks as follows:

```
Trace: File started downloading: 0 /myprefix/file1.img
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Trace: File finished downloading: 468021 /myprefix/file1.img Download Speed: 179.236 Kilobit/s in 468021 ms
```

You might now wonder: Why is the throughput only 179 Kilobit/s, when the link can handle up to 1 Megabit/s? The reason for this is that we only issued an Interest for a fragment once the previous Interest (for a fragment of the manifest) was answered. This results in lot of un-used capacity. The solution to this is pipe-lining Interests, which we also implemented (see next sub-section).


### Pipe-Lining Interests (Enhanced File Consumer)
While the basic FileConsumer works for testing, we recommend using the enhanced version, ```FileConsumerCbr```, (Cbr = Constant bit rate), which issues pipelines Interests to fully utilize the link capacity the consumer has. This is achieved by calculating the average number of data packets per second that this consumer can handle (read: ```LinkDownloadSpeedInBytes/PayloadSize```). In addition, we also issue Interests for File/1, File/2, ... before we receive an answere for File/Manifest. While this could lead to unnecessary Interests, in most cases it will speed up the file transfer (as the files transferred are large enough).

Using the Enhanced File Consumer is as simple as replacing ``ns3::ndn::FileConsumer`` with ``ns3::ndn::FileConsumerCbr``, as shown in the following code:

```cplusplus
  // Consumer
  ndn::AppHelper consumerHelper("ns3::ndn::FileConsumerCbr");
  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file1.img"));
```

You can find the full example here [AMuSt-ndnSIM/examples/ndn-file-simple-example3-enhanced.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/blob/master/examples//ndn-file-simple-example3-enhanced.cpp).

The output shows a throughput of 961 Kilobit/s and looks like this:
```
Trace: File started downloading: 0 /myprefix/file1.img
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Trace: File finished downloading: 87285 /myprefix/file1.img Download Speed: 961.06 Kilobit/s in 87285 ms
Simulation Finished.
```
This number is still a bit lower than 1 Megabit/s. As the ndn header uses 51 of our available 1500 bytes (Payload size 1449), the resulting goodput is roughly 96.6%  (roughly 966 Kilobit/s), which we almost manage to achieve.

As you can see, ``FileConsumerCbr`` performs much better, and it is the recommended Consumer to use. In addition, ``FileConsumerCbr`` also handles timeouts, re-transmissions and out-of-order packets.


### Testing File Transfers with Real Data
With FileConsumer and FileProducer we are transfering real payloads (data packets) over the simulated network. For some scenarios it might be necessary to also use the file transfered via the simulator (e.g., in a consumer app). We added a method for writeing the received file to the storage with the following command:
```cplusplus
  consumerHelper.SetAttribute("WriteOutfile", StringValue("/home/username/somefile.img"));
```
Feel free to also compare the outfile with the original file by using ```m5sum``` or ```sha1sum```.

While this method helps for debugging/testing, it is not recommended storing those files on permanent storage for long running simulations. Instead, use a ramdisk. In addition, writing files to storage (or ramdisk) does impact performance negatively. 


### Tracing Several File Consumers 
When considering a larger scenario with many clients, it might be required to log the trace output to a file for all clients. This can be easily achieved by using the ``FileConsumerLogTracer`` with the following line of code:
```
  ndn::FileConsumerLogTracer::InstallAll("file-consumer-log-trace.txt");
```

An example of this is provided in [AMuSt-ndnSIM/examples/ndn-file-simple-example4-multi.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/tree/master/examples/ndn-file-simple-example4-multi.cpp). The output produced by this scenario looks like this:
```
Trace: File started downloading: 0 /myprefix/file1.img
Trace: File started downloading: 0 /myprefix/file1.img
Trace: File started downloading: 0 /myprefix/file1.img
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Received Manifest! FileSize=10485760, MaxPayload=1449
Resulting Max Seq Nr = 7237
Trace: Manifest received: 41 /myprefix/file1.img File Size: 10485760
Trace: File finished downloading: 90485 /myprefix/file1.img Download Speed: 927.072 Kilobit/s in 90485 ms
Trace: File finished downloading: 90687 /myprefix/file1.img Download Speed: 925.007 Kilobit/s in 90687 ms
Trace: File finished downloading: 90691 /myprefix/file1.img Download Speed: 924.966 Kilobit/s in 90691 ms
Simulation Finished.
```
As you can see, it becomes harder to track what happens. Needless to say, because of NDNs interest aggregation and caching mechanisms the speed is almost 1 Mbit/s for all three clients.

We can now open file-consumer-log.trace.txt which looks as follows:
```
Time	Node	AppId	InterestName	Event	Info
0	0	0	/myprefix/file1.img	DownloadStarted	NoInfo
0	1	0	/myprefix/file1.img	DownloadStarted	NoInfo
0	2	0	/myprefix/file1.img	DownloadStarted	NoInfo
0.041786	0	0	/myprefix/file1.img	ManifestReceived	FileSize=10485760
0.041786	1	0	/myprefix/file1.img	ManifestReceived	FileSize=10485760
0.041786	2	0	/myprefix/file1.img	ManifestReceived	FileSize=10485760
90.4855	1	0	/myprefix/file1.img	DownloadFinished	Speed=905.343;Time=90.485
90.6877	0	0	/myprefix/file1.img	DownloadFinished	Speed=903.327;Time=90.687
90.6918	2	0	/myprefix/file1.img	DownloadFinished	Speed=903.287;Time=90.691
```

This file provides similaro utput as the console, but it's in a more "processable" format (CSV with \t as separator).



### Hosting Virtual Files
The next step for a larger simulation would be creating multiple producers that host different content with different file sizes. Eventually this will lead to a point where one needs to have multiple directories with multiple files and various sizes on a storage device. While this is a waste of storage (for simulatons), it also slows down simulations (File I/O is THE bottleneck for simulations). 

To counter this problem we implemented a so called ``FakeFileServer``, which takes an additional parameter (``MetaDataFile`` - a CSV file with filenames and filesizes in bytes, separated by a comma) and stores that information (filename + size) in memory, instead of reading actual files from a (slow) storage device.


Here we provide an example of this CSV file, containing 3 files at various sizes:
```
file1.img,1000000
file2.img,1500000
file3.img,2000000
```
Assuming this file is called fake.csv, the producer and consumer configuration will look like this:
```cplusplus
  // Consumer
  ndn::AppHelper consumerHelper("ns3::ndn::FileConsumerCbr");
  
  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file1.img"));
  consumerHelper.Install(nodes.Get(0));

  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file2.img"));
  consumerHelper.Install(nodes.Get(1));

  consumerHelper.SetAttribute("FileToRequest", StringValue("/myprefix/file3.img"));
  consumerHelper.Install(nodes.Get(2));

  ...

  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FakeFileServer");

  // Producer will reply to all requests starting with /prefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("MetaDataFile", StringValue("fake.csv"));
  producerHelper.Install(nodes.Get(4)); // install to some node from nodelist
```
The full example is available here: [AMuSt-ndnSIM/examples/ndn-file-simple-example5-virtual.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/tree/master/examples/ndn-file-simple-example5-virtual.cpp).


### Summary
In this chapter, we have presented the basics of file transfers in AMuSt-ndnSIM. We can now use this for implementing adaptive multimedia streaming (which is the main purpose of this simulator).

------------------
## 2. Hosting Multimedia Files
For multimedia streaming, we use the concept of Dynamic Adaptive Streaming over HTTP, adapted for the file transfer logic from Part 1. We make use of an extended version of bitmovins [libdash](https://github.com/bitmovin/libdash), which is an open source library for MPEG-DASH. Our version, [AMuSt-libdash](https://github.com/ChristianKreuzberger/AMuSt-libdash/) includes a dummy multimedia player, video playback buffer and several adaptation logics.

Before we talk any more about the client, we describe the process of hosting DASH multimedia content. Hosting DASH content is as simple as telling the ``FileServer`` (see previous chapter) to host a directory with the DASH content. We can basically use any MPEG-DASH compatible video stream, as long as the Media Presentation Description (MPD) file can be parsed by libdash (Note: we only support BASIC MPD structures, with only one period, and no wildcards). 


Alternatively, content can also be hosted by using the ``FakeFileServer`` and a .csv file containing all names and filesizes of all multimedia files. However, the MPD file still needs to be provided by ``FileServer`` (can not be done with ``FakeFileServer``). This process has the advantage of still using a realistic dataset but avoiding the overhead of storing multiple gigabyte of (unnecessary) data on permanent storage.


Yet another method of hosting DASH content is achieved by providing only the information of which representations the video should have. This is handled by ``FakeMultimediaServer``. The big disadvantage of this approach is that all segments of one representation have the exact same size, something that is not necessarily true in real world multimedia applications. However, this method has almost no storage overhead and is most likely the easiest one to start with.

In the following we will show examples for all three approaches, starting with the easiest one (``FakeMultimediaServer``), followed by ``FileServer`` and ``FakeFileServer``.


### Hosting Virtual Content using ``FakeMultimediaServer``
Netflix has [announced](http://techblog.netflix.com/2015/12/per-title-encode-optimization.html) which adaptive streaming representations they have been using in the past in a [blog post](http://techblog.netflix.com/2015/12/per-title-encode-optimization.html). For convenience, here they are:

```
reprId,screenWidth,screenHeight,bitrate
1,320,240,235
2,384,288,375
3,512,384,560
4,512,384,750
10,640,480,1050
11,720,480,1750
20,1280,720,2350
21,1280,720,3000
30,1920,1080,4300
31,1920,1080,5800
``` 

The convenient thing about ``FakeMultimediaServer`` is that we only need to provide those (purposely CSV-formatted) data and some more in a CSV file (netflix_vid1.csv):
```
segmentDuration=2
numberOfSegments=1800
reprId,screenWidth,screenHeight,bitrate
1,320,240,235
2,384,288,375
3,512,384,560
4,512,384,750
10,640,480,1050
11,720,480,1750
20,1280,720,2350
21,1280,720,3000
30,1920,1080,4300
31,1920,1080,5800
```

Config: ``segmentDuration`` specifies the duration of a segment in seconds (2 is a kind-of industry-standard) and ``numberOfSegments`` specifies the number of segments for a video. In this case, the length of the video would be 3600 seconds (60 minutes). The CSV format below contains the representation identifier (reprId, integer), screen width / height (integer), and a bitrate in kbit/s (integer). 

If you want to have several videos with various lengths, you need to provide several CSV files. For convenience, we have uploaded example files for [Netflix here](../representations/). Last but not least, we need to configure a server, which is shown in the code example below:
```cplusplus
  ndn::AppHelper fakeDASHProducerHelper("ns3::ndn::FakeMultimediaServer");

  // This fake multimedia producer will reply to all requests starting with /myprefix/FakeVid1
  fakeDASHProducerHelper.SetPrefix("/myprefix/FakeVid1");
  fakeDASHProducerHelper.SetAttribute("MetaDataFile", StringValue("representations/netflix_vid1.csv"));
  // We just give the MPD file a name that makes it unique
  fakeDASHProducerHelper.SetAttribute("MPDFileName", StringValue("vid1.mpd"));

  fakeDASHProducerHelper.Install(nodes.Get(0));

  // We can install more then one fake multimedia producer on one node:

  // This fake multimedia producer will reply to all requests starting with /myprefix/FakeVid2
  fakeDASHProducerHelper.SetPrefix("/myprefix/FakeVid2");
  fakeDASHProducerHelper.SetAttribute("MetaDataFile", StringValue("representations/netflix_vid2.csv"));
  // We just give the MPD file a name that makes it unique
  fakeDASHProducerHelper.SetAttribute("MPDFileName", StringValue("vid2.mpd"));


  // This fake multimedia producer will reply to all requests starting with /myprefix/FakeVid3
  fakeDASHProducerHelper.SetPrefix("/myprefix/FakeVid3");
  fakeDASHProducerHelper.SetAttribute("MetaDataFile", StringValue("representations/netflix_vid3.csv"));
  // We just give the MPD file a name that makes it unique
  fakeDASHProducerHelper.SetAttribute("MPDFileName", StringValue("vid3.mpd"));

```

You can find the full example, including clients (which we will discuss later), in TODO.


### Hosting Actual DASH Content with ``FileServer``
While there are plenty of programs out there to create MPEG-DASH streams, reproducability of demos and simulations is a key problem. Therefore we recommend using existing datasets, and we refer to the following two datasets for MPEG-DASH Videos:

* [DASH/AVC Dataset](http://www-itec.uni-klu.ac.at/dash/?page_id=207) by Lederer et al. [[PDF]](http://www-itec.uni-klu.ac.at/bib/files/p89-lederer.pdf) [[Bib]](http://www-itec.uni-klu.ac.at/bib/index.php?key=Mueller2012&bib=itec.bib) [[Website]](http://www-itec.uni-klu.ac.at/dash/?page_id=207) (we recommend using the older 2012 version for testing purpose)
* [DASH/SVC Dataset](http://concert.itec.aau.at/SVCDataset/) by Kreuzberger et al. [[PDF]](http://www-itec.uni-klu.ac.at/bib/files/dash_svc_dataset_v1.05.pdf) [[Bib]](http://www-itec.uni-klu.ac.at/bib/index.php?key=Kreuzberger2015a&bib=itec.bib) [[Website]](http://concert.itec.aau.at/SVCDataset/)

In the following, we will provide an example for multimedia streaming with DASH/AVC and DASH/SVC, based on those two datasets. We selected the Big Buck Bunny movie to achieve compareable results.

#### DASH/AVC
**Note:** This will require roughly 4 Gigabyte of diskspace
We are going to download the BigBuckBunny movie from the DASH/AVC Dataset, segment length 2 seconds.
First of all, download the video files from [here](http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/):
```bash
cd ~
mkdir -p multimediaData/AVC/BBB/
cd multimediaData/AVC/BBB/

wget -r --no-parent --cut-dirs=5 --no-host --reject "index.html" http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/
```
this folder should also contain the MPD file ``BigBuckBunny_2s_isoffmain_DIS_23009_1_v_2_1c2_2011_08_30.mpd``, if not, you can download the
 [MPD file](http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/BigBuckBunny_2s_isoffmain_DIS_23009_1_v_2_1c2_2011_08_30.mpd) manually.
```bash
wget http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/BigBuckBunny_2s_isoffmain_DIS_23009_1_v_2_1c2_2011_08_30.mpd
```
Open the MPD in an editor of your choice, and locate the ``<BaseURL>`` tag. Change
```xml
<BaseURL>http://www-itec.uni-klu.ac.at/ftp/datasets/mmsys12/BigBuckBunny/bunny_2s/</BaseURL>
```
to
```xml
<BaseURL>/myprefix/AVC/BBB/</BaseURL>
```
Furthermore, we recommend renaming the file to a name of your choice, e.g., BBB-2s.mpd.
```bash
mv BigBuckBunny_2s_isoffmain_DIS_23009_1_v_2_1c2_2011_08_30.mpd BBB-2s.mpd
```

Last but not least, we would like to separate the mpd file from the segments. Therefore we move the mpd file up by one folder (to ~/multimediaData/AVC/):

```bash
mv BBB-2s.mpd ../
```

Finally, your directories should look like this:

 * multimediaData/AVC/
    * BBB-2s.mpd
    * BBB/
        * bunny_2s_50kbit/*.m4s
        * bunny_2s_100kbit/*.m4s
        * ...
        * bunny_2s_8000kbit/*.m4s

#### DASH/SVC
**Note:** This will require roughly 1 Gigabyte of diskspace
We are going to download the BigBuckBunny movie from the DASH/SVC Dataset, with a segment length of 2 seconds, and no temporal scalability. First of all, download the video files from [here](http://concert.itec.aau.at/SVCDataset/dataset/BBB/III/segs/).
```bash
cd ~
mkdir -p multimediaData/SVC/BBB/
cd multimediaData/SVC/BBB/
mkdir III
cd III
wget -r --no-parent --no-host-directories --no-directories http://concert.itec.aau.at/SVCDataset/dataset/BBB/III/segs/1080p/
```
Second, download the [MPD file](http://concert.itec.aau.at/SVCDataset/dataset/mpd/BBB-III.mpd).
```bash
wget http://concert.itec.aau.at/SVCDataset/dataset/mpd/BBB-III.mpd
```
Open the MPD in an editor of your choice, and locate the ``<BaseURL>`` tag. Change
```xml
<BaseURL>http://concert.itec.aau.at/SVCDataset/dataset/BBB/III/segs/1080p/</BaseURL>
```
to
```xml
<BaseURL>/myprefix/SVC/BBB/III/</BaseURL>
```

#### Creating a Server with Above Datasets
Finally, creating a server is as simple as:
```cplusplus
  // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /myprefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("ContentDirectory", StringValue("/home/username/multimediaData"));
  producerHelper.Install(nodes.Get(0)); // install to some node from nodelist

```


See [examples/ndn-multimedia-avc-server.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/blob/master/examples/ndn-multimedia-avc-server.cpp) for the full example for AVC. 

### Hosting Real Content using FakeFileServer
This part assumes that you followed the DASH Dataset part just before this section. You probably noticed that a good portion of storage space is gone. To counter this problem, we provide ``FakefileServer`` with a CSV file for BBB (AVC), which you can get [here](../datasets/segmentlist/). This file contains a list of all segments and their size. We generated this list by first downloading the dataset and then using some Linux command line magic (mainly sed and awk):
```shell
find . -type f -exec du -a {} + | sed 's/[ \t]/,/g' | sed 's/\.\///g' | awk -F $',' ' { t = $1; $1 = $2; $2 = t; print; } ' OFS=$',' > list_of_files.csv
```

Now we need to configure two producers as follows

 * ``FileServer`` hosts the actual MPD file of BBB at /myprefix/AVC
 * ``FakeFileServer``hosts the virtual segments of BBB at /myprefix/AVC/BBB


```cplusplus
  // Producer responsible for hosting the MPD file
  ndn::AppHelper mpdProducerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /myprefix/AVC/ and hosts the mpd file there
  mpdProducerHelper.SetPrefix("/myprefix/AVC");
  mpdProducerHelper.SetAttribute("ContentDirectory", StringValue("/home/someuser/multimediaData"));
  mpdProducerHelper.Install(nodes.Get(0)); // install to some node from nodelist

  // Producer responsible for hosting the virtual segments
  ndn::AppHelper fakeSegmentProducerHelper("ns3::ndn::FakeFileServer");

  // Producer will reply to all requests starting with /myprefix/AVC/BBB/ and hosts the virtual segment files there
  fakeSegmentProducerHelper.SetPrefix("/myprefix/AVC/BBB");
  fakeSegmentProducerHelper.SetAttribute("MetaDataFile", StringValue("dash_dataset_avc_bbb.csv"));
  fakeSegmentProducerHelper.Install(nodes.Get(0)); // install to some node from nodelist
```

See [examples/ndn-multimedia-avc-fake-server.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/blob/master/examples/ndn-multimedia-avc-fake-server.cpp) for the full example.






## 3. Using the Multimedia Consumer

In this section we will talk about the multimedia consumer and how to use it, with both AVC and SVC content.

### AVC Content
Our multimedia consumers are built on top of the FileConsumers, therefore it is necessary to specify which FileConsumer you want. We recommend using the ``FileConsumerCbr`` class, hence you should use the following code for requesting AVC content:
```cplusplus
  ns3::ndn::AppHelper consumerHelper("ns3::ndn::FileConsumerCbr::MultimediaConsumer");
  consumerHelper.SetAttribute("AllowUpscale", BooleanValue(true));
  consumerHelper.SetAttribute("AllowDownscale", BooleanValue(false));
  consumerHelper.SetAttribute("ScreenWidth", UintegerValue(1920));
  consumerHelper.SetAttribute("ScreenHeight", UintegerValue(1080));
  consumerHelper.SetAttribute("StartRepresentationId", StringValue("auto"));
  consumerHelper.SetAttribute("MaxBufferedSeconds", UintegerValue(30));
  consumerHelper.SetAttribute("StartUpDelay", StringValue("0.1"));

  consumerHelper.SetAttribute("AdaptationLogic", StringValue("dash::player::RateAndBufferBasedAdaptationLogic"));
  consumerHelper.SetAttribute("MpdFileToRequest", StringValue(std::string("/myprefix/AVC/BBB/BBB-2s.mpd" )));

  ApplicationContainer app1 = consumerHelper.Install (nodes.Get(2));
```

See [examples/ndn-multimedia-simple-avc-example1.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/blob/master/examples/ndn-multimedia-simple-avc-example1.cpp) for the full example.


### SVC Content
This is very similar to the AVC case, you just need to specify a different adaptation logic (and the correct MPD file):

```cplusplus
  ns3::ndn::AppHelper consumerHelper("ns3::ndn::FileConsumerCbr::MultimediaConsumer");
  consumerHelper.SetAttribute("AllowUpscale", BooleanValue(true));
  consumerHelper.SetAttribute("AllowDownscale", BooleanValue(false));
  consumerHelper.SetAttribute("ScreenWidth", UintegerValue(1920));
  consumerHelper.SetAttribute("ScreenHeight", UintegerValue(1080));
  consumerHelper.SetAttribute("StartRepresentationId", StringValue("auto"));
  consumerHelper.SetAttribute("MaxBufferedSeconds", UintegerValue(30));
  consumerHelper.SetAttribute("StartUpDelay", StringValue("0.1"));

  consumerHelper.SetAttribute("AdaptationLogic", StringValue("dash::player::SVCBufferBasedAdaptationLogic"));
  consumerHelper.SetAttribute("MpdFileToRequest", StringValue(std::string("/myprefix/SVC/BBB/BBB-III.mpd" )));

  ApplicationContainer app1 = consumerHelper.Install (nodes.Get(2));
```


See [examples/ndn-multimedia-simple-svc-example1.cpp](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM/blob/master/examples/ndn-multimedia-simple-svc-example1.cpp) for the full example.


## Multimedia Consumers Options
Our Multimedia Consumers have plenty of options to be configured. First of all, the two most important ones are 

 * ``AdaptationLogic`` 
 *  ``MpdFileToRequest``

``MpdFileToRequest`` is, like the name suggests, the MPD file of the video to play. For the purpose of having a quicker start-up, it is possible to use gzip'ed MPD files here, by specifying a file like this: ``mpdfilename.mpd.gz``(the file obviously needs to be available in the directory).
``AdaptationLogic``  depends heavily on the MPD file. If the MPD file contains SVC content, the following adaptation logics are available: 

 * ``SVCBufferBasedAdaptationLogic`` - Buffer based adaptation logic for SVC, based on BIEB (Sieber et al., [Implementation and User-centric Comparison of a Novel Adaptation Logic for DASH with SVC](http://ieeexplore.ieee.org/xpls/abs_all.jsp?arnumber=6573184&tag=1)) with alpha=8 and gamma=16 - make sure to set ``MaxBufferedSeconds`` to at least ``gamma + 2 * alpha + 1`` (in this case 33), we recommend setting it to at least 60
* ``SVCBufferBasedAdaptationLogicAggressive`` - same as above, but with alpha=2 and gamma=8, we recommend setting ``MaxBufferedSeconds`` to at least 30
* ``SVCBufferBasedAdaptationLogicNormal``- same as above, but with alpha=4 and gamma=8, we recommend setting ``MaxBufferedSeconds`` to at least 30
 * ``SVCRateBasedAdaptationLogic``
 * ``SVCNoAdaptationLogic`` - requests all SVC representations, starting from the base layer, until the segment needs to be consumed
 * ``AlwaysLowestAdaptationLogic`` (default value) - use always the lowest representation available (warning: this might not work in all cases for SVC content)

For AVC:

 * ``AlwaysLowestAdaptationLogic`` (default value) - use always the lowest representation available
 * ``RateBasedAdaptationLogic``- the estimated throughput is the main deciding factor for the representation used
 * ``RateAndBufferBasedAdaptationLogic`` - the clients local buffer and the estimated throughput will be used for determining the representation
 * ``DASHJS`` - TODO


Then we have several options that the clients can use for their "simulated screens":

 * ``ScreenWidth`` and ``ScreenHeight``
 * ``AllowUpscale`` and ``AllowDownscale``

Those 4 options are used to determine which representations from the MPD file are unusable for the client. For example, if you have a mobile phone with a 1280x720 screen, you should not request a representation with 1920x1080 and downscale the video.
Instead, you can request the 640x320 representation and upscale it to 1280x720 or request the 1280x720 representation and use it without scaling. For some scenarios it might even be necessary to disable upscaling too.


 * ``MaxBufferedSeconds`` (default: 30)
 * ``StartUpDelay``(default: 2.0)

Those options are used to determine the maximum amount of buffered seconds. While on most computers, this number could be rather large, especially with mobile phones and tablets this number is restricted by the available memory. Common practice for those is around 30 seconds.
The ``StartUpDelay`` is used for scenarios, where you want the client to buffer first, and start playing after a certain amount of time. This could beneficial for scenarios where you have a slow Internet connection, as it is better to buffer first, and then play. If set to 0, the player will start playing the video as soon as the first segment has been successfully downloaded.

 * ``StartRepresentationId`` (default: "auto")

Can take the following values: "lowest", "auto" (means: use adaptation logic) or a certain representation id. This attribute is for testing purpose, and we recommend using "auto".

## Multimedia Consumers and Tracers
For evaluation purpose, we have a special tracer for multimedia consumers available. This tracer logs the following events:

 * Segment consumed at which time with representation id and representation bitrate
 * Video playback stall (freeze)
 * Start-up delay

Example:
```cplusplus
// include the header file
#include "ns3/ndnSIM/utils/tracers/ndn-dashplayer-tracer.hpp"

// ...
int
main(int argc, char* argv[])
{
  // ...
  // install the tracer
  ndn::DASHPlayerTracer::InstallAll("dash-output.txt");

  Simulator::Run();
  Simulator::Destroy();
  // ...
}
```

The output will be a CSV file called dash-output.txt, and look like this (for SVC content)
```
Time  Node  SegmentNumber   SegmentDuration(sec)  SegmentRepID SegmentBitrate(bit/s)  StallingTime(msec) SegmentDepIds
0.42821 2       0                 2                        0               624758                  0
2.42821 2       1                 2                        0               624758                  0
...
18.4282 2       9                 2                        0               624758                  0
20.4282 2       10                2                        1               2122081                 0       0
...
26.4282 2       13                2                        1               2122081                 0       0
28.4282 2       14                2                        2               5108358                 0       0,1
30.4282 2       15                2                        2               5108358                 0       0,1
32.4282 2       16                2                        2               5108358                 0       0,1
34.4282 2       17                2                        2               5108358                 0       0,1
36.4282 2       18                2                        3               9885231                 0       0,1,2
38.4282 2       19                2                        3               9885231                 0       0,1,2
...
```
AVC content will look similar, but will not have the SegmentDepIds filled at all.

See examples/ndn-multimedia-simple-avc-example2-tracer.cpp and See examples/ndn-multimedia-simple-svc-example2-tracer.cpp for the full sourcecode.

### Other Tracers
For evaluation purposes most other tracers should also work, for instance, Content Store Tracer. Please note, that due to some limitations with ns-3/ndnSIM, the AppId of the MultimediaConsumer will change over time, therefore you need to filter the Node (```NodeId```), instead of ```AppId``` in all traces.

See [examples/ndn-multimedia-simple-avc-example2-tracers.cpp](examples/ndn-multimedia-simple-avc-example2-tracers.cpp) and [examples/ndn-multimedia-simple-svc-example2-tracers.cpp](examples/ndn-multimedia-simple-svc-example2-tracers.cpp) for the full example.

------------------

## 3. Building Large Networks with BRITE and Installing Multimedia Clients
## Basics
The [BRITE Network Generator](http://www.cs.bu.edu/brite/) is an open source project, which can be used by ndnSIM/ns-3. We have set up a wrapper class (see [helper/ndn-brite-topology-helper.cpp](helper/ndn-brite-topology-helper.cpp) and  [helper/ndn-brite-topology-helper.hpp](helper/ndn-brite-topology-helper.hpp)). All you need is a brite config file.



```cplusplus
// include the header file
#include "ns3/ndnSIM/helper/ndn-brite-topology-helper.hpp"
// ...

  // Create Brite Topology Helper
  ndn::NDNBriteTopologyHelper bth ("brite.conf");
  bth.AssignStreams (3);
  // tell the topology helper to build the topology
  bth.BuildBriteTopology ();

// ...

```

You can find the full example later. 

## Random Network
Use the ``--RngRun=`` option of [ns-3 Random Variables](https://www.nsnam.org/docs/manual/html/random-variables.html#id1) to generate different instances of the random network.

Example:
```bash
./waf --run "ndn-multimedia-brite-example1 --RngRun=0" --vis
./waf --run "ndn-multimedia-brite-example1 --RngRun=1" --vis
./waf --run "ndn-multimedia-brite-example1 --RngRun=2" --vis
```


## Installing the NDN Stack, Multimedia Clients, Routing, ...
First, we include the ndn brite helper and configure the application to have a parameter for the brite config file:

```cplusplus
#include "ns3/ndnSIM/helper/ndn-brite-helper.hpp"
```

```cplusplus
int
main(int argc, char* argv[])
{
  std::string confFile = "brite.conf";

  CommandLine cmd;
  cmd.AddValue ("briteConfFile", "BRITE configuration file", confFile);
  cmd.Parse(argc, argv);
```

Next, we create an instance of the NDN stack and build the topology:

```cplusplus
  // Create NDN Stack
  ndn::StackHelper ndnHelper;

  // Create Brite Topology Helper
  ndn::NDNBriteTopologyHelper bth (confFile);
  bth.AssignStreams (3);
  // tell the topology helper to build the topology 
  bth.BuildBriteTopology ();
```

Now we go through that topology and create a ```NodeContainer``` for servers, clients and routers, so we can distingiush the configuration of those nodes.

```cplusplus
  // Separate clients, servers and routers
  NodeContainer client;
  NodeContainer server;
  NodeContainer router;
```

Iterate over all AS (Autonomous System) and get all non leaf nodes
```cplusplus
  for (uint32_t i = 0; i < bth.GetNAs(); i++)
  {
    std::cout << "Number of nodes for AS: " << bth.GetNNodesForAs(i) << ", non leaf nodes: " << bth.GetNNonLeafNodesForAs(i) << std::endl;
    for(int node=0; node < bth.GetNNonLeafNodesForAs(i); node++)
    {
      std::cout << "Node " << node << " has " << bth.GetNonLeafNodeForAs(i,node)->GetNDevices() << " devices " << std::endl;
      //container.Add (briteHelper->GetNodeForAs (ASnumber,node));
      router.Add(bth.GetNonLeafNodeForAs(i,node));
    }
  }
```

Iterate over all AS and get all leaf nodes
```cplusplus
  uint32_t sumLeafNodes = 0;
  for (uint32_t i = 0; i < bth.GetNAs(); i++)
  {
    uint32_t numLeafNodes = bth.GetNLeafNodesForAs(i);

    std::cout << "AS " << i << " has " << numLeafNodes << "leaf nodes! " << std::endl;

    for (uint32_t j= 0; j < numLeafNodes; j++)
    {
      if (decideIfServer(i,j)) // some decision, can be random
      {
        server.Add(bth.GetLeafNodeForAs(i,j));
      }
      else
      {
        client.Add(bth.GetLeafNodeForAs(i,j));
      }
    }

    sumLeafNodes+= numLeafNodes;
  }

  std::cout << "Total Number of leaf nodes: " << sumLeafNodes << std::endl;

```
Where ``decideIfServer(asNumber, leafNumber)`` needs to have some logic to decide whether this node should be a client or a server. Generally, a random strategy works well, you just need to make sure you know how many servers and clients you want to have (in relation).


Now let's isntall the ndn stack and content stores:
```cplusplus
  // clients do not really need a large content store, but it could be beneficial to give them some
  ndnHelper.SetOldContentStore ("ns3::ndn::cs::Stats::Lru","MaxSize", "100");
  ndnHelper.Install (client);

  // servers do not need a content store at all, they have an app to do that
  ndnHelper.SetOldContentStore ("ns3::ndn::cs::Stats::Lru","MaxSize", "1");
  ndnHelper.Install (server);

  // what really needs a content store is the routers, which we don't have many
  ndnHelper.setCsSize(10000);
  ndnHelper.Install(router);
```
and choose a forwarding strategy
```cplusplus
  // Choosing forwarding strategy
  ndn::StrategyChoiceHelper::InstallAll("/myprefix", "/localhost/nfd/strategy/best-route");
```

Install the GlobalRoutingHelper
```cplusplus
  // Installing global routing interface on all nodes
  ndn::GlobalRoutingHelper ndnGlobalRoutingHelper;
  ndnGlobalRoutingHelper.InstallAll();
```

Install the Multimedia Consumer (in this case an SVC consumer)
```cplusplus
  // Installing multimedia consumer
  ns3::ndn::AppHelper consumerHelper("ns3::ndn::FileConsumerCbr::MultimediaConsumer");
  consumerHelper.SetAttribute("AllowUpscale", BooleanValue(true));
  consumerHelper.SetAttribute("AllowDownscale", BooleanValue(false));
  consumerHelper.SetAttribute("ScreenWidth", UintegerValue(1920));
  consumerHelper.SetAttribute("ScreenHeight", UintegerValue(1080));
  consumerHelper.SetAttribute("StartRepresentationId", StringValue("auto"));
  consumerHelper.SetAttribute("MaxBufferedSeconds", UintegerValue(30));
  consumerHelper.SetAttribute("StartUpDelay", StringValue("0.1"));

  consumerHelper.SetAttribute("AdaptationLogic", StringValue("dash::player::SVCBufferBasedAdaptationLogic"));
  consumerHelper.SetAttribute("MpdFileToRequest", StringValue(std::string("/myprefix/SVC/BBB/BBB-III.mpd" )));
```

Start clients / install logic
```cplusplus
  // Randomize Client File Selection
  Ptr<UniformRandomVariable> r = CreateObject<UniformRandomVariable>();
  for(int i=0; i<client.size (); i++)
  {
    // TODO: Make some logic to decide which file to request
    consumerHelper.SetAttribute("MpdFileToRequest", StringValue(std::string("/myprefix/SVC/BBB/BBB-III.mpd" )));
    ApplicationContainer consumer = consumerHelper.Install (client[i]);

    std::cout << "Client " << i << " is Node " << client[i]->GetId() << std::endl;

    // Start and stop the consumer
    consumer.Start (Seconds(1.0)); // TODO: Start at randomized time
    consumer.Stop (Seconds(600.0));
  }
```

Install the servers
```cplusplus
   // Producer
  ndn::AppHelper producerHelper("ns3::ndn::FileServer");

  // Producer will reply to all requests starting with /myprefix
  producerHelper.SetPrefix("/myprefix");
  producerHelper.SetAttribute("ContentDirectory", StringValue("/home/someuser/multimediaData/"));
  producerHelper.Install(server); // install to servers

  ndnGlobalRoutingHelper.AddOrigins("/myprefix", server);
```

Install Routing:
```cplusplus
  // Calculate and install FIBs
  ndn::GlobalRoutingHelper::CalculateAllPossibleRoutes();
```

Run the Simulation
```cplusplus
  Simulator::Stop(Seconds(1000.0));
  Simulator::Run();
  Simulator::Destroy();

  std::cout << "Simulation ended" << std::endl;

  return 0;
}
```


Full example here: [examples/ndn-multimedia-brite-example1.cpp](examples/ndn-multimedia-brite-example1.cpp).
Brite config File: TODO

------------------



