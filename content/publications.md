+++
title = "📚 Publications"
menu = "main"
+++

## Most important academic papers

* [Slither: A Static Analysis Framework for Smart Contracts](https://arxiv.org/abs/1908.09878) (Josselin Feist, **Gustavo Grieco**, Alex Groce)

* [Manticore: A User-Friendly Symbolic Execution Framework for Binaries and Smart Contracts](https://arxiv.org/abs/1907.03890) (Mark Mossberg, Felipe Manzano, Eric Hennenfent, Alex Groce, **Gustavo Grieco**, Josselin Feist)

* [SMARTIAN: Enhancing Smart Contract Fuzzing with Static and Dynamic Data-Flow Analyses](https://islab-sogang.github.io/data/ase2021.pdf) (Jaeseung Choi, Doyeon Kim, Soomin Kim, **Gustavo Grieco**, Alex Groce, S. Cha)

* [Optimizing Seed Selection for Fuzzing](https://www.usenix.org/system/files/conference/usenixsecurity14/sec14-paper-rebert.pdf) (Alexandre Rebert, Sang Kil Cha, Thanassis Avgerinos, Jonathan Foote, David Warren, **Gustavo Grieco**, David Brumley)

* [Undangle: early detection of dangling pointers in use-after-free and double-free vulnerabilities](https://software.imdea.org/~juanca/papers/undangle_issta12_av.pdf)
  (Juan Caballero, **Gustavo Grieco**, Mark Marron, Antonio Nappa)

* [Echidna: effective, usable, and fast fuzzing for smart contracts](https://agroce.github.io/issta20.pdf) (**Gustavo Grieco**, Will Song, Artur Cygan, Josselin Feist, Alex Groce)

* [Toward Large-Scale Vulnerability Discovery using Machine Learning](https://www.covert.io/research-papers/deep-learning-security/Toward%20large-scale%20vulnerability%20discovery%20using%20Machine%20Learning.pdf) (**Gustavo Grieco**, G. Grinblat, Lucas C. Uzal, Sanjay Rawat, Josselin Feist, L. Mounier)

* [QuickFuzz: an automatic random fuzzer for common file formats](https://people.kth.se/~buiras/publications/QFHaskell2016.pdf) (**Gustavo Grieco**, Martín Ceresa, Pablo Buiras)

The complete list is [available in Semantic Scholar](https://www.semanticscholar.org/author/Gustavo-Grieco/39340848).

## Blog posts

* [Improving the state of Cosmos Fuzzing](https://blog.trailofbits.com/2024/02/05/improving-the-state-of-cosmos-fuzzing/)
* [An Echidna for all seasons](https://blog.trailofbits.com/2020/03/30/an-echidna-for-all-seasons/)

## CVEs

A number of [CVEs](https://en.wikipedia.org/wiki/Common_Vulnerabilities_and_Exposures) accumulated over years of using fuzzers:

### Using [QuickFuzz](https://people.kth.se/~buiras/publications/QFHaskell2016.pdf):

* Mozilla Firefox (CVE-2016-1933, CVE-2015-7194, CVE-2015-7216, CVE-2015-7217)
* webkit (CVE-2016-9642, CVE-2016-9643)
* libxml2 (CVE-2016-3627, CVE-2016-4483)
* gdk-pixbuf (CVE-2015-7552, CVE-2015-7674, CVE-2015-7673)
* mujs (CVE-2016-9109)
* gif2webp (CVE-2016-9085)
* jq (CVE-2016-4074)
* jasson (CVE-2016-4425)
* libgd (CVE-2016-6132, CVE-2016-6214)
* graphicsmagick (CVE-2016-2317, CVE-2016-2318)
* mini-XML (CVE-2016-4570, CVE-2016-4571)
* vlc (CVE-2016-3941)
* cpio (CVE-2016-2037)
* cairo (CVE-2016-9082)

### Using [American Fuzzy Lop](https://github.com/google/AFL):

* Expat XML (CVE-2016-0718)
* libxml2 (CVE-2016-4483, CVE-2016-3627, CVE-2015-8035)
* Apache xerces-C (CVE-2016-0729, CVE-2016-2099)