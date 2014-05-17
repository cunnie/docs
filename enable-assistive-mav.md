### ShiftIt Unbound <sup>[[1]](#prometheus)</sup>

#### Or, *Use the CLI to grant Accessibility Privileges to Mavericks Applications*

[ShiftIt](https://github.com/onsi/ShiftIt) <sup>[[2]](#shiftit)</sup> is a utility we use at Pivotal Labs.  It has special security requirements that make it difficult to install without user intervention. This blog post describes the techniques we used to determine what we needed to do in order to satisfy those security requirements without requiring user intervention during the install process.

### Problem Statement

At Pivotal Labs we use an unattended run of [Sprout Wrap](https://github.com/pivotal-sprout/sprout-wrap) to pre-install a common base of applications (e.g. ShiftIt, Google Chrome, Mozilla Firefox, JetBrains RubyMine) on a developer workstation before turning it over to the developer (we don't want our developers to waste time installing software; we want them to be able to begin coding as soon as they sit down).

Prior to OS X Mavericks, Apple's security mechanism was coarse:  to enable ShiftIt (which needed to be able to intercept keystrokes), one merely enabled Assistive Devices (**System Preferences &rarr; Accessibility &rarr; Enable access for assistive devices**). But this had a drawback: not only did this allow ShiftIt to control your computer, but it opened the floodgates for *any* application to control your computer.

Apple addressed this shortcoming by introducing a finer-grained approach to security. In Mavericks, one enables access *by individual application*; e.g. unlike OS X Mountain Lion, enabling ShiftIt's to control your computer did *not* allow any other application to control your computer.



---

<a name="prometheus"><sup>1</sup></a> with apologies to [Percy Bysshe Shelley](http://en.wikipedia.org/wiki/Prometheus_Unbound_%28Shelley%29)

<a name="shiftit"><sup>2</sup></a> ShiftIt is a fork of a fork of an OS X application, currently maintained by [Onsi Fakhouri](https://github.com/onsi); it controls window placement via keyboard shortcuts.