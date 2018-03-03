%-----------------------------------------------------------------------------
%
%               Template for sigplanconf LaTeX Class
%
% Name:         sigplanconf-template.tex
%
% Purpose:      A template for sigplanconf.cls, which is a LaTeX 2e class
%               file for SIGPLAN conference proceedings.
%
% Guide:        Refer to "Author's Guide to the ACM SIGPLAN Class,"
%               sigplanconf-guide.pdf
%
% Author:       Paul C. Anagnostopoulos
%               Windfall Software
%               978 371-2316
%               paul@windfall.com
%
% Created:      15 February 2005 
%
%-----------------------------------------------------------------------------


\documentclass[nocopyrightspace]{sigplanconf}

% The following \documentclass options may be useful:

% preprint      Remove this option only once the paper is in final form.
% 10pt          To set in 10-point type instead of 9-point.
% 11pt          To set in 11-point type instead of 9-point.
% authoryear    To obtain author/year citation style instead of numeric.

\usepackage{amsmath}
\usepackage{paralist}

\begin{document}

\special{papersize=8.5in,11in}
\setlength{\pdfpageheight}{\paperheight}
\setlength{\pdfpagewidth}{\paperwidth}

\conferenceinfo{CONF 'yy}{Month d--d, 20yy, City, ST, Country} 
\copyrightyear{20yy} 
\copyrightdata{978-1-nnnn-nnnn-n/yy/mm} 
\doi{nnnnnnn.nnnnnnn}

% Uncomment one of the following two, if you are not going for the 
% traditional copyright transfer agreement.

%\exclusivelicense                % ACM gets exclusive license to publish, 
                                  % you retain copyright

%\permissiontopublish             % ACM gets nonexclusive license to publish
                                  % (paid open-access papers, 
                                  % short abstracts)

\titlebanner{banner above paper title}        % These are ignored unless
\preprintfooter{short description of paper}   % 'preprint' option specified.

\title{Pytheus: An Easily Programmable Heterogeneous Computing System}
%\subtitle{Subtitle Text, if any}

\authorinfo{Stefan B. O'Neil}
           {North Carolina State University}
           {soneil@ncsu.edu}
%\authorinfo{Name2\and Name3}
%           {Affiliation2/3}
%           {Email2/3}

\maketitle

\begin{abstract}
Solid-state drives have entered the mainstream of commercial storage devices. These drives come equipped with their own processors and memory to provide the necessary support for the flash translation layer. Numerous solutions exist for enabling the host machine to leverage these in-storage computational resources, making possible considerable performance gains. However, these existing approaches rely heavily on device- code running on the SSD, making them inflexible for adaptation to new applications, and inaccessible to application developers writing in modern, high-level programming languages. This paper proposes to redress this weakness by moving the task of code generation for the SSD to the host compiler. We target Pyston, a just-in-time Python compiler, and use built-in Python functionality to share relevant data structures and machine code directly with the SSD. [rest of abstract will deal with results]
\end{abstract}

%\category{CR-number}{subcategory}{third-level}

% general terms are not compulsory anymore, 
% you may leave them out
%\terms
%term1, term2

%\keywords
%keyword1, keyword2

\section{Introduction}

In keeping with the Von Neumann model of computer architecture, computer systems have traditionally separated computation, performed by a dedicated processor, from data storage, typically accomplished by a hard disk drive (HDD), or more recently, a solid state drive (SSD). Over the past few decades, processor performance has improved dramatically. Storage access latency has not enjoyed similar improvement.

The result of this mismatch is that storage access has become the most significant limiting factor for computing performance. This is particularly true for data intensive applications, which have become ever more prevalent with the advent of big data.

In the last few years, SSDs have entered the commercial mainstream of storage devices. SSDs offer significant performance improvement in access latency over HDDs. However, as flash-based devices, SSDs also differ from HDDs in a number of use-relevant ways. Unlike HDDs, SSDs have different granularity for reads and writes, cannot write to dirty pages before clearing them, and have lifetime limits on number of writes per block before a given block becomes unusable. These differences necessitate different access patterns from those appropriate for HDDs. To work seamlessly with the HDD-optimized storage interfaces common among host machines, SSDs must therefore perform additional logic known as the flash translation layer (FTL) to process traditional read-write requests from the host. To accomplish this, SSDs come equipped with their own internal processors and DRAM.

The additional computational resources in peripheral devices such as SSDs create an opportunity to bypass the bottleneck of data transfer by moving away from the classical von Neumann separation of computation and data storage. By moving some of the computational tasks in a program to near-data processors such as those found in SSDs, a more distributed model of computing can reduce the traffic between the central processing unit and peripheral devices.

The project undertaken in this paper is to adapt a proven programming model for in-storage processing called Morpheus for compatibility and ease of use with a high level programming language. We target the Python programming language, primarily for its simplicity, ease of use, and high popularity among application developers.

Our programming model consists of the application runtime environment, which shares relevant data structures and compiled SSD-compatible machine code with a kernel-space memory buffer, which the host then shares with the SSD. After the SSD executes this machine code and stores the results back into its own memory buffer, these are then copied directly back into the host-side memory buffer via Direct Memory Access.

The remainder of this paper is broken down into the following sections:\\
    
\begin{compactenum}
	\setcounter{enumi}{1}
	\item Heterogeneous Computing Models
 	\item In-storage Processing
 	\item Just-in-Time Compilation
	\item Detailed System Overview
	\item Generation of Machine Code from Pyston
	\item Results
	\item Related Work
	\item Conclusion
\end{compactenum}

\section{Heterogeneous Computing Models}

The proliferation of auxiliary processing units in modern computing systems has spurred a huge amount of research interest into the potential performance improvements such systems provide. These auxiliary processing units include a wide array of different technologies. The most common of these are graphical processing units (GPUs), field programmable gate arrays (FPGAs), and in-storage processors. Within each of these types of technologies, there is a great deal of variability in the specific properties a given device may have. Different GPUs, for example, employ a wide variety of different memory hierarchies. Furthermore, there are many other types of auxiliary processing units that a computing system may employ, apart from the common three listed above. Prominent examples include the tensor processing unit developed by Google, vision processing units, and physics processing units.

One intuitive way to exploit the performance advantages of heterogeneous computing systems is to apply these diverse computational resources to application code. Many auxiliary processing units are designed specifically to handle types of computations that are difficult for traditional CPUs to perform efficiently, such as graphics rendering or the training of neural networks. FPGAs can be configured to perform operations common to a particular application much faster than a traditional CPU is capable of.

In addition to the speed advantage of specialized processors, a heterogeneous computing model can also improve application performance by moving the most data-intensive computations of an application closer to the data storage. In this way, auxiliary processing resources can act as a filter for data passing from the storage device to the main processor, reducing traffic across the main bus.

Database applications, with their high volume of data access, are a prime candidate for acceleration by means of heterogeneous computing systems. Several projects have proposed using FPGAs to implement database operations (list of citations). In 2014, Woods et al. demonstrated that FPGAs could be used to implement not only basic tasks such as projection and selection-based filtering, but also more complex SQL operators such GROUP-BY and WHERE. Since then, various others have extended this approach with systems that move an even greater portion of database computations from the host processor down to FPGA pipelines. Ziener et al. demonstrate how far this approach can be taken in their paper, “FPGA-Based Dynamically Reconfigurable SQL Query Processing”, in which they describe and prototype a system in which a pipeline of FPGAs can be reconfigured to completely handle any query from a library of modules covering all major SQL operations.

One complicating factor for sharing information between different components of a heterogeneous system is that of mapping between addresses in separate address spaces. Unifying the address space of heterogeneous components with that of the host memory could allow for more seamless data transfer. The project FlashTier uses this technique as part of an architecture for using a flash-based storage device as a cache for a higher capacity storage drive. For the sake of facilitating this design strategy, Olson et al. propose an interface for coherence messages between the host and auxiliary hardware accelerators called Crossing Guard.

\subsection{In-storage Processing}

Of particular relevance for our research is the use of processors built into storage devices as components of a heterogeneous system. All modern SSDs contain contain a built-in processor and accompanying DRAM memory cache. These are necessary for SSDs to transparently map logical block addresses from the host to physical locations within the flash memory, and to perform flash management algorithms such as wear leveling and garbage collection. These tasks are collectively known as the Flash Translation Layer (FTL).

If made available to the host, SSD processor time and memory space not utilized for FTL can be applied to other tasks. The opportunity is especially great because of two advantages that these in-storage processors enjoy:

1. lower power consumption than typical host processors

2. lower latency and higher bandwidth accessing storage, without adding traffic to the main bus

One very popular area of application for in-storage computing is big data analytics. Big data applications are heavily bottlenecked by storage bandwidth. Because in-storage processors can access this data before it reaches the main bus however, using in-storage processing to perform initial filtering of the data has the potential to greatly reduce traffic between the host and the storage device. Choi and Kee demonstrate the potential benefits of this storage-level data filtering using model analysis in "Energy Efficient Scale-In Clusters with In-Storage Processing for Big-Data Analytics". Aside from performance improvements, they also demonstrate the potential for reducing energy consumption. Their findings indicate that energy consumption is dominated by the cost of transferring data between storage and host, meaning that lowering this traffic is likely to deliver significant energy cost reduction in addition to performance improvement. Do et al. also examine the potential of in-storage processing to improve performance of database applications in their paper, “Query Processing on Smart SSDs: Opportunities and Challenges”. To aid in this analysis, they implemented some basic database operations on an SSD, and modified an SQL database management system to offload those operations to the SSD. Kang et al. take a similar approach in “Enabling Cost-effective Data Processing with Smart SSD”, targeting the Hadoop MapReduce framework for optimization instead.

Jun et al. implement a system called BlueDBM which introduces a number of optimizations for a distributed, big-data-oriented system built on SSDs. These optimizations include both in-storage processing, implemented with an FPGA at the storage device level, and a global address space for the distributed system. They are able to demonstrate meaningful performance gains by use of their in-storage processing engine.

The Willow project focuses more exclusively on leveraging in-storage processing for application code. Willow consists of a system driver, a SSD with a custom interface, and a template for user programs called SSD-Apps, which handles many of the complexities of writing in-storage code for the user. They demonstrate the performance benefits of this approach with six of SSD-Apps of their own, which show marked performance improvements over an implementation not utilizing in-storage processing power.

The Biscuit project also exposes in-storage computational resources to the programmer, but does so in a more user-friendly manner. Specifically, Gu et al. develop a framework based on the flow-based programming model that supports C++ programming and another of other important features, such as multithreading and dynamic loading and unloading of tasks onto the SSD.

The work that most directly inspired the current project is described in a paper titled, “Morpheus: Creating Application Objects Efficiently for Heterogeneous Computing”. Dr. Tseng et al present a model of computation called Morpheus, in which the host machine compiles and transmits code to the SSD, which then processes data and transmits the results back to a corresponding program running on the host machine, generated by the same compiler. By targeting the data-intensive task of deserialization – specifically, of ASCII stored values into C-compatible integer arrays – their Morpheus-SSD implementation reduced context switching by an average of 98\% in the host machine, energy consumption by upwards of 42\%, and execution time by varying but significant margins on a wide range of applications.

The primary drawback of the Morpheus project is its reliance on the programmer to manually write code intended for execution on the SSD. Code to be executed on the SSD must be written in C, and is exceptionally challenging to debug. These factors make Morpheus less flexible and programmer-friendly than is practical for adoption by mainstream application developers.

\subsection{Host-SSD Interface and Communication}

Because 


\section{In-storage Processing}

\subsection{Big Data Analytics}




\section{Just-in-Time Compilation}

Just-in-time (JIT) compilation is a method of code execution in which portions of program code are compiled or recompiled during runtime. JIT compilation is highly compatible with dynamic languages such as Ruby and Python, but also offers strong performance advantages over pure interpretation. Because of the popularity of high level dynamic programming languages, JIT compilers for such languages provide a useful environment for making cutting edge advances in systems technologies more accessible to general application developers.

\subsection{JIT Code Generation for Heterogeneous Systems}

Leveraging the performance advantages of heterogeneous systems requires writing code for auxiliary processing units. Coding for these heterogeneous computing resources often requires detailed hardware knowledge, and the use of specific low-level programming languages such as OpenCL. This is a significant barrier for application developers trying to leverage the advantages of heterogeneous computing systems in their own programs.

Using JIT compilers to generate the appropriate code for the target heterogeneous processor instead can remove this burden from the programmer. The compatibility of JIT compilers with dynamic programming languages means that this approach can bring direct, native control of heterogeneous computing resources to such languages.

\subsection{JIT Compiler Projects for Python}

In its default implementation, Python is an interpreted language. The performance limitations of line-by-line interpretation have inspired a handful of high profile projects to re-implement the language with a JIT compiler.

\section{Detailed System Overview}

\section{Generation of Machine Code from Pyston}

\section{Results}

\section{Related Work}

\section{Conclusion}

%\appendix
%\section{Appendix Title}

%This is the text of the appendix, if you need one.

%\acks

%Acknowledgments, if needed.

% We recommend abbrvnat bibliography style.

\bibliographystyle{abbrvnat}

% The bibliography should be embedded for final submission.

\begin{thebibliography}{}
\softraggedright

\bibitem[Smith et~al.(2009)Smith, Jones]{smith02}
P. Q. Smith, and X. Y. Jones. ...reference text...

\end{thebibliography}


\end{document}

%                       Revision History
%                       -------- -------
%  Date         Person  Ver.    Change
%  ----         ------  ----    ------

%  2013.06.29   TU      0.1--4  comments on permission/copyright notices
