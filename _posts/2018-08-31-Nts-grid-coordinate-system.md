# Working on library to handle survey systems of Western Canada

Working with Oil and Gas well location data in Western Canada is challenging due to the prevelance of many different identifier types that are used to represent the location of a given well, we have Dls, Nts & Uwi. I developed a library that contains types that represent the various grid systems. These types perform validation of input values to ensure that the values are valid. The types can also perform conversion between geographic locations and the survey system locations.

## Dominion Land Survey (DLS)

The dominion land survey (DLS) is a survey system that was developed before GPS coordinates. It is comprised by repeatedly surveying 1x1 square mile plots of land called sections.

Alberta, Saskatchewan, and parts of Manitoba and parts of British Columbia are mapped on a grid system into townships of approximately 36 square miles (6 mi. x 6mi.).  Each township consists of 36 sections (1 mi. x 1 mi). Each section is further divided into 16 legal subdivisions (LSDs). The numbering system for sections and LSDs uses a back-and-forth system where the numbers may be increasing either to the right or the left in the grid. Since the DLS system is based on actual survey data, there can be gaps in the coverage.

### Breakdown of DLS Identifier

 Given the location: _04-11-082-04W6_
| Part | Value |
|---|---|
| Legal Sub division | 04|
| Section | 11|
| Township | 082|
| Range | 04|
| Meridian | W6|

#### Legal Sub division (LSD)

A legal subdivision is the smallest unit in the DLS identifier, It is used to divide the Section into 16 pieces of ~1/4 mi x 1/4 mi or ~40 acres. These are numbered from 1-16 in a zig zag pattern.
| | | | |
|-|-|-|-|
13|14|15|16
12|11|10|09
05|06|07|08
04|03|02|01

#### Section

A section is a 1x1 mile parcel, it divides the township into 36 pieces. Sections are labelled from 1-36 with a zig zag pattern.
| | | | | | |
|-|-|-|-|-|-|
31|32|33|34|35|36
30|29|28|27|26|25
19|20|21|22|23|24
18|17|16|15|14|13
07|08|09|10|11|12
06|05|04|03|02|01

#### Township

A Township is a 6x6 parcel it is numbered from south to north (1-126) and is located at the intersection of a range line and township line.

#### Correction Lines

Because the sections are surveyed as squares a number of corrections to the survey data must be made as you travel north. These are the correction lines, and they are caused by resetting the range numbering along baselines. This results in a jog of the range road, the magnitude of this jogging increases as you move further west from any meridian. This jogging occurs every 4 townships as you move north, with the first correction at 12.075 miles north of the baseline and the next at 36.225 miles north of the baseline.

#### Ranges

Ranges are numbered from each meridian and increase as you move west. For locations that are west of the prime meridian the ranges may number as high as 34 at the lower latitudes. For all other meridians the range

#### Township Lines

Township lines are numbered from the base meridian @ 49 degrees (Majority of Canada-US border). Township liness run east-west and are spaced at 6.0375 miles to allow for the 6 miles of a township and the 3 road allowances of one chain each (~20 meters). In addition, there are baselines that repeat every 24.15 miles or 4 township lines.

#### Meridians

Meridians, the first or prime meridian was established just west of winnipeg. There are a total of 8 western meridians as shown below. However the library only maps sections in meridians 1-6.

| Longitude | Name |
|---|---|
| 97° 27' 28.4" | W1M (a.k.a Prime Meridian) |
| 102° | W2M |
| 106° | W3M |
| 110° | W4M (Sask-Alberta Border) |
| 114° | W5M |
| 118° | W6M |
| 122° | W7M |
| 122° 45' 39.6" | W8M (Coast Meridian) |

### Compression of DLS Section Markers

One of the problems that comes up when working with the DLS Survey system is that the townships are not referenced to geographic coordinates. In order to tell if a given location is within a coordinate you must lookup the survey data for the township and then perform a geometry calculation to determine if a point is within the boundary of that township. This means that any library that can be used to convert between geographic coordinates and the DLS survey must have access to all of the known township boundary points to be effective.

There are more than 15,000 townships of 6x6 miles across western canada.
Most townships have a total of 36 sections with 4 markers defined for each of the corners. This gives 144 coordinates for every township.

One of the goals of this project was to build a library that could be used without a dependency on a database engine. So I thought about all of the ways that the storage of this information could be made most efficient so that it could be included as a resource section within the library directly.

There are more than 2.1 million locations that we need to know to perform coordinate conversions. Each of the 2.1 million points needs to record both a latitude and longitude as a pair of 32 bit floating point numbers. In addition we need to record a key value that is comprised of a township number, range number and meridian. If we use an 8 bit integer for each of these key fields then we end up with a row size of 88 bits, 64 bits of data + 24 bits of key. We can further optimize the storage of the key field, consider that the maximum range of the township is 127, in binary we only require 7 bits to represent this number. Additionally our ranges top out at 34, which requires 6 bits. Meridians go from 1 - 6 and can be represented in 3 bits. So we only require `7 + 6 + 3 = 16` bits to represent our key. While a one byte saving doesn't sound like much in this case it represents 2MB of data. This gives us a new row size of 80 bits.

Example of bit stuffing method to generate our key value

```c#
var key = (ushort)((uint)(meridian << 13 | range << 7) | township);
```

### Township block encoding

You may be wondering why we don't encode the section numbers in our township, I chose to write the township blocks using a fixed layout starting at section 1 and moving to section 36, each section writes the coordinates using a fixed encoding of [se,sw,ne,nw]. This method allows us to use a simple offset calculation to move to the coordinate that we require. For example to lookup the NorthEast coordinate of section 27 we use an offset of `((27-1) * 8) + 6 = 214.`

### Further Compression

At this point our dataset is near 20MB in total, to further reduce the information we employ a Deflate stream encoder. The compression ratio of the Deflate is very good due to the highly repetitive nature of the values passed to it. From the Deflate encoding we get a dataset of ~4.7MB. This is small enough that I feel we can embed the data into a resource section of the library.

## British Columbia Geographic System

The British Columbia Geographic System (BCGS) Sometimes this is refered to as NTS - National Topographic System, the system upon which BCGS is based. This system is aligned with geographic coordinates and as such it is trivial to convert between geographic coordinates and BCGS. Identifiers are written with the following convention : _B-001-L/093-F-10_
Where 'B' is the quarter unit, '001' is the unit, 'L' is the block, 093 is the series, F is the map area, and 10 is the sheet. Note that the string after the '/' character is the same code that would be used in an Nts survey identifier to represent a map at 1:20,000 scale.

### Based on National Topographic System

Locations throughout all of Canada can be specified using the National Topographic System (NTS), as it is a system based on lines of latitude and longitude rather than recorded survey data. It is used extensively in British Columbia to mark well and pipeline locations. The exception in BC to this system is the Peace River Block, which is surveyed using the DLS system.

### Breakdown of BC NTS identifier

#### Series

The NTS system consists of many series (sometimes called maps), identified by series numbers that increase by 1 for the next series to the north, and increase by 10 for the next series to the west. Most series are 8 degrees across and 4 degrees high.
The province of British Colombia is mapped within the following series
| | | | |
|-|-|-|-|
|114|104|94|
| |103|93|83
| |102|92|82

#### Map Areas

Each series is subdivided into 16 areas, which are given letters from A to P, starting at the southeast corner and then labelled with a back and forth lettering with A starting at the South East Corner. Each map area is 2 degrees of longitude (width) and 1 degrees of latitude (north south).
| | | | |
|-|-|-|-|
  M|N|O|P
  L|K|J|I
  E|F|G|H
  D|C|B|A

#### Sheets

Each area is then subdivided into 16 sheets (sometimes called a "map sheet") which are given numbers from 1 to 16. The back-and-forth method is used for numbering, starting in the southeast corner.Each sheet is 0.5 degrees of longitude (width) and 0.25 degrees of latitude (north south)
| | | | |
|-|-|-|-|
13|14|15|16
12|11|10|09
05|06|07|08
04|03|02|01

#### Blocks

A sheet is subdivided into 12 blocks, 4 across and 3 high. These are given letters to identify them, from A to L, using the back-and-forth method starting in the southeast corner. Each block is 0.125 degrees of longitude (width) and (0.25/3) degrees of latitude (north south)
| | | | |
|-|-|-|-|
L|K|J|I
E|F|G|H
D|C|B|A

#### Units

Each block is subdivided into 100 "units" numbered as 1 to 10, from east to west in the southernmost row, and 11 to 20 from east to west in the next row up, and so on.Note the direction of the numbers is always towards the left.
| | | | | | | | | | |
|-|-|-|-|-|-|-|-|-|-|
20|19|18|17|16|15|14|13|12|11
10|09|08|07|06|05|04|03|02|01

#### Quarters

Finally, each unit is subdivided into 4 quarter units labeled A, B, C, and D, starting in the southeast and moving clockwise.
  C|D
  B|A

Anyhow, the project files can be downloaded from [GitHub](https://github.com/RaysceneNS/GisLibrary)