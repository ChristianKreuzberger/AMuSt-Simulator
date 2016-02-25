# Adaptive Multimedia Streaming Simulator Framework

This is the main repository for all AMuSt (Adaptive Multimedia Streaming) repositories. The following repositories are currently available:

 * [AMuSt-libdash](https://github.com/ChristianKreuzberger/AMuSt-libdash) - an extended version of bitmovin's libdash library
 * [AMuSt-ns3](https://github.com/ChristianKreuzberger/AMuSt-ns3) - a modified version of Network Simulator 3 (ns3) for AMuSt
 * [AMuSt-ndnSIM](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM) - a modified version of Named Data Networking Simulator (ndnSIM) for AMuSt


## Installation Instructions
Installation instructions can be found in the respective repositories:

 * [AMuSt-ns3](https://github.com/ChristianKreuzberger/AMuSt-ns3) - a modified version of Network Simulator 3 (ns3) for AMuSt
 * [AMuSt-ndnSIM](https://github.com/ChristianKreuzberger/AMuSt-ndnSIM) - a modified version of Named Data Networking Simulator (ndnSIM) for AMuSt

## Tutorials

One of the goals was to make sure that the difference between writing scenario-code is as similar as possible for both, ndnSIM and ns-3. We have prepared two tutorials, one for ns-3 and one for ndnSIM, which are very similar, but address the specifics of the respective simulation environment.

 * [AMuSt-ns3 Tutorial](tutorials/tutorial_amust_ns3.md) 
 * [AMuSt-ndnSIM Tutorial](tutorials/tutorial_amust_ndnsim.md) 

In addition, we are providing more information about the DASH adaptation logics we use and how they work in the 

 * [AMuSt-libdash Tutorial](https://github.com/ChristianKreuzberger/AMuSt-libdash/blob/master/tutorial.md).


## Info about libdash
We are using a custom version of libdash [AMuSt-libdash](https://github.com/ChristianKreuzberger/AMuSt-libdash), so please make sure you use the version provided in the tutorial above.


## What is MPEG-DASH?
MPEG-DASH (ISO/IEC 23009-1:2012) is a standard for adaptive multimedia streaming over HTTP connections, which is 
adapted for NDN file-transfers in this project. For more information about MPEG-DASH, please consult the following
links:

* [DASH at Wikipedia](http://en.wikipedia.org/wiki/Dynamic_Adaptive_Streaming_over_HTTP)
* [Slideshare: DASH - From content creation to consumption](http://de.slideshare.net/christian.timmerer/dynamic-adaptive-streaming-over-http-from-content-creation-to-consumption)
* [DASH Industry Forum](http://dashif.org/)



## Citation
We are currently working on a technical report/paper. For now, you can cite it by using the following text:

*Christian Kreuzberger, Daniel Posch, Hermann Hellwagner "AMuSt Framework - Adaptive Multimedia Streaming Simulation Framework for ns-3 and ndnSIM", https://github.com/ChristianKreuzberger/AMust-Simulator/ *


## Acknowledgements
This work was partially funded by the Austrian Science Fund (FWF) under the CHIST-ERA project [CONCERT](http://www.concert-project.org/) 
(A Context-Adaptive Content Ecosystem Under Uncertainty), project number I1402.

