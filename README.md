# OGC API - Coverages

This GitHub repository contains [OGC](http://opengeospatial.org)'s
standard for querying geospatial information on the web, "OGC API - Coverages".

OGC API standards define modular API building blocks to spatially enable Web
APIs in a consistent way. [OpenAPI](http://openapis.org) is used to define the
reusable API building blocks with responses in JSON and HTML.

The OGC API family of standards is organized by resource type. OGC API -
Coverages specifies the fundamental API building blocks for interacting with
coverages. The spatial data community uses the term 'coverage' for homogeneous
collections of values located in space/time, such as spatio-temporal sensor,
image, simulation, and statistics data.

If you are unfamiliar with the term 'coverage', the explanations on
the [Coverages DWG Wiki](http://myogc.org/go/coveragesDWG) provide more detail and links to educational material.
Additionally, [Coverages: describing properties that vary with location (and time)](https://www.w3.org/TR/sdw-bp/#coverages)
in the W3C/OGC Spatial Data on the Web Best Practice document may be considered.

## Overview

### General

This [OGC API - Coverages](https://github.com/opengeospatial/ogc_api_coverages) specification establishes how to access coverages as defined by the [Coverage Implementation Schema (CIS) 1.1](http://docs.opengeospatial.org/is/09-146r6/09-146r6.html) through [OpenAPI](https://www.openapis.org/).

Current scope:

* Only gridded coverages are addressed, not MultiPoint/Curve/Surface/SolidCoverages. Reason is that gridded coverages receive most attention today.
* Only CIS 1.1 GeneralGridCoverage is addressed, other coverage types will follow later. Reason is to have a first version early which shows and allows to evaluate the principles.
* Only coverage extraction functionality is considered, not general processing (as is provided with Web Coverage Service (WCS) extensions such as the Processing Extension). Exceptions from this rule are subsetting including band subsetting, scaling, and CRS conversion and data format encoding, given their practical relevance.
* Subsetting is considered in the query component only for now. As typically all dimensions in a coverage are of same importance subsetting might not fit perfectly in the hierarchical nature of the path component. Further, subsetting may reference any axis and leave out any other, which makes positional parameters unsuitable. Nevertheless subsetting in the path component particularly limited to fixed subsets might be considered in a future version. The principles of the `/tiles` path defined in OGC API - Maps & Tiles might be a good fit and shall be explored.

As such, the functionality provided below resembles [OGC Web Coverage Service (WCS) 2.1 Interface Standard - Core](http://docs.opengeospatial.org/is/17-089r1/17-089r1.html).

### Background Information: WCS Service Model

OGC WCS 2 has an internal model of its storage organization based on which the classic operations GetCapabilities, DescribeCoverage, and GetCoverage can be explained naturally.
This model consists of a single CoverageOffering resembling the complete WCS data store. It holds some service metadata describing service qualities (such as WCS extensions, encodings, CRSs, and interpolations supported, etc.). At its heart, this offering holds any number of OfferedCoverages. These contain the coverage payload to be served, but in addition can hold coverage-specific service-related metadata (such as the coverage's Native CRS).

![WCS 2 internal storage model](http://external.opengeospatial.org/twiki_public/pub/CoveragesDWG/CoveragesBigPicture/WCS-internal-organization.png)

Discussion has shown that the OpenAPI functionality also assumes underlying service and object descriptions, so a convergence seems possible. In any case, it will be advantageous to have a similar "mental model" of the server store organization on hand to explain the various functionalities introduced below.

## Principles

[OpenAPI](https://www.openapis.org/) establishes URL-based access patterns, as defined by [RFC 3986 "Uniform Resource Identifier (URI): Generic Syntax"](https://tools.ietf.org/html/rfc3986), [RFC 3987 "Internationalized Resource Identifiers (IRIs)"](https://tools.ietf.org/html/rfc3987), and [RFC 6570 "URI Template"](https://tools.ietf.org/html/rfc6570) following a syntax like
        http://www.acme.com/path/to/resource/{id}{?query,parameters}
whereby

* `/path/to/resource/{id}` defines the local path to the resource to be retrieved on the node identified by the head part, http://www.acme.com.
* the `{?query,parameters}` represent key/value pairs with further parameters passed to the request.

As a general rule,

* accessing a coverage component is done in the path section,
* subsetting is done in the query parameter section
* format encoding is controlled via HTTP headers

As the coverage model normatively is given by the corresponding XML schema (JSON and RDF are built in sync with XML) specification of the [OpenAPI](https://www.openapis.org/) access paths follows this schema. Note, though, that OWS Common, while normatively
referenced in CIS, is not followed by [OpenAPI](https://www.openapis.org/), so here deviations will occur.

For path expressions abbreviations (i.e., aliases) may be defined for convenience.

## [OGC API - Coverages](https://github.com/opengeospatial/ogc_api_coverages) Requests

This clause defines the basics of accessing OGC coverages via OpenAPI.

### General

Requirement:
A service offering [OGC API - Coverages](https://github.com/opengeospatial/ogc_api_coverages) access shall be accessible via an URL as service endpoint (subsequently referred to as Base URL).

Example:
        http://acme.com/oapi

Requirement:
Coverage access paths, as defined in the next clause, are concatenated to the Base URL via a "/" character, followed (optionally) by a query part.

Example:
        http://acme.com/oapi/collections/{collectionid}/coverage/rangeset

A request may be denied for server-specific reasons, such as quota.

### Coverage Access Paths

In this clause, the path component of [OGC API - Coverages](https://github.com/opengeospatial/ogc_api_coverages) access is established.

Access paths follow the XML Schema of [CIS](http://docs.opengeospatial.org/is/09-146r6/09-146r6.html) in their structure.

To access coverage service constituents, such as formats supported, OGC 14-121 Web Query Service provides guidance, see https://github.com/opengeospatial/ogc_api_coverages/blob/master/CIS%2BWCS-standards/14-121_Web-Query-Service_2016-06-19.pdf .

### Coverage Query Parameters

In this clause, the query component of [OGC API - Coverages](https://github.com/opengeospatial/ogc_api_coverages) access is established.

Subsetting parameters may be mixed in a query part, in no particular order.

#### Coverage Subsetting

Without any subsetting parameter the whole coverage extent is the target resource addressed. If a subsetting operation is provided then the coverage subset indicated is the target resource addressed.

Coverage subsetting is indicated through the SUBSET parameter name. The value following the "=" symbol is built as follows:

    SubsetSpec:            SUBSET = axisName ( intervalOrPoint )
    axisName:              {NCName}
    intervalOrPoint:       interval | point
    interval:              low ,  high
    low:                   point | *
    high:                  point | *
    point:                 {number} | " {text} "    //" = double quote = ASCII code 0x42

where {NCName} is an XML-style identifier not containing ":" (colon) characters, {number} is an integer or floating-point number, and {text} is some general ASCII text (such as a time and date notation in ISO 8601). A coverage access request may have any number of SUBSET parameters, however for each axis of the coverage at most one SUBSET parameter may be provided.

In case of an interval a trim operation is specified, with lower and upper bound. In case of a point a slicing operation is specified. For the detailed semantics of subsetting, trimming, and slicing see [OGC WCS](http://docs.opengeospatial.org/is/17-089r1/17-089r1.html).

### Coverage Encoding

If no format encoding is specified in the request then a representation of the coverage shall be returned in its Native Format.

In general, file formats are not always capable of representing all coverage information. This is one reason why applications may prefer receiving a coverage in some different format.

Note:

* The notion of Native Format refers to the range set only. Returning a coverage in this format may mean that some coverage constituents cannot be represented appropriately, and consequently will be missing from the coverage result.

An application may request a particular format encoding through one of the following two options:

* By indicating the format's MIME type identifier in the Accept and Accept-Encoding HTTP header of the request. The syntax for format encoding HTTP headers is defined in [RFC 7231 "Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content"](https://tools.ietf.org/html/rfc7231).
* By appending a query parameter `f=m` where `m` is the format's MIME type identifier.

If both options are present simultaneously in the request then the `f` parameter shall have preference.

If the format chosen is not capable of representing the coverage data requested this shall lead to a request error.

Note:

* Extensions may provide further options, such as full content negotiation as per the HTTP standard.

For the detailed semantics of format encoding see [OGC WCS](http://docs.opengeospatial.org/is/17-089r1/17-089r1.html).

## Examples

This section contains examples for coverage access, subsetting, and encoding. It assumes a service endpoint of http://acme.com/oapi/ .

### Service Metadata

The first part is about service metadata. Currently this is wild speculation as it will mainly be defined through [OGC API - Common](https://github.com/opengeospatial/oapi_common).

* http://acme.com/oapi/servicedescription/formatssupported  -- returns a list of formats in which coverage representations can be requested

### Coverage Finding

* http://acme.com/oapi/collections  -- returns a list of all collection identifiers
* http://acme.com/oapi/collections?bbox=160.6,-55.95,-170,-25.89  -- returns a list of all collection identifiers intersecting with the New Zealand economic zone (any time, any elevation, etc.); the CRS in which the bbox parameters are expressed is WGS84, defined in OGC API - Common and likely in a future OGC API - Catalog
* http://acme.com/oapi/collections/{collectionid}  -- returns the description of a specific collection including links to available sub-paths like /coverage, /tiles, or /map
* http://acme.com/oapi/collections/{collectionid}/coverage  -- returns a general description of the coverage represented by collectionid containing the coverage's envelope (the full domainset might be huge and thus is omitted here), rangetype, and service metadata like the coverage's native format

Notes:

* all list results include links (cf. OGC API - Common)
* "coverage" is a specialization of "items"; further names could be defined in future (RectifiedGridCoverage, features, ...)
* "description of collection" to be clarified (defined in OGC API - Common?)
* bbox is OGC API - Common syntax
  * in OGC API - Coverages Core, no subsetting on vertical and time coordinates
  * horizontal coordinates: all filtering is evaluated in WGS84 (may require bounding box translation before intersecting)

### Coverage Access

The second part is about coverage access, which (as described earlier) is driven by the coverage structure and, hence, given:

* http://acme.com/oapi/collections/{collectionid}/coverage/description -- returns the whole coverage description consisting of domainset, rangetype, and metadata (but not the rangeset)
* http://acme.com/oapi/collections/{collectionid}/coverage/domainset  -- returns the coverage's domain set definition
* http://acme.com/oapi/collections/{collectionid}/coverage/rangetype  -- returns the coverage's range type information (i.e., a description of the data semantics)
* http://acme.com/oapi/collections/{collectionid}/coverage/metadata  -- returns the coverage's metadata (may be empty)
* http://acme.com/oapi/collections/{collectionid}/coverage/rangeset  -- returns the coverage's range set, i.e., the actual values in the coverage's Native Format (see format encoding for ways to retrieve in specific formats)
* http://acme.com/oapi/collections/{collectionid}/coverage/all  -- returns all of the above namely the coverage's domainset, rangetype, metadata, and rangeset comparable to a GetCoverage response

### Coverage Subsetting

The third part is about query parameters:

* http://acme.com/oapi/collections/{collectionid}/coverage?SUBSET=Lat(40,50)&SUBSET=Long(10,20)  -- returns a coverage cutout between (40,10) and (50,20), as multipart coverage
* http://acme.com/oapi/collections/{collectionid}/coverage/rangeset?SUBSET=Lat(40,50)&SUBSET=Long(10,20)  -- returns a coverage cutout between (40,10) and (50,20), in the coverage's Native Format
* http://acme.com/oapi/collections/{collectionid}/coverage?SUBSET=time("2019-03-27")  -- returns a coverage slice at the timestamp given (in case the coverage is Lat/Long/time the result will be a 2D image)

### Metadata and Coverage encoding

When not specified otherwise, the metadata will be returned in CIS XML format where 
applicable. Alternatively when the `Accept` header includes `application/json` at a
higher relevance than `text/xml` then the CIS JSON encoding will be applied.

Similarly, coverages are always returned in their native format (even when 
subsetted) unless the `Accept` header is to set and another acceptable format is in
the list. Additionally, encoding specific parameters can be passed as-well using 
mime type parameters.

For example, the following request expresses the intent to download the rangeset in
a tiled and LZW-compressed TIFF file, but allows a fallback to a simple image/tiff 
and NetCDF encoding.

```http
Accept: image/tiff; tiling=yes; tilewidth=128; tileheight=256; compress=LZW, image/tiff, application/netcdf
```

When using the TIFF encoding format the available parameters are documented in the
[WCS 2.0 Specification - GeoTIFF Coverage Encoding Profile](https://portal.opengeospatial.org/files/?artifact_id=54813).
It is not required to use the `GEOTIFF:` prefix for each parameter.

When the full coverage access (metadata + data) is accessed, the encoding of the 
parts can be controlled by mime type parameters too. Each part of the coverage 
(domainset, rangetype, metadata, rangeset, etc.) is controllable by a mime type 
parameter of the same name. Each parameter can contain a list of formats, just as
using the `Accept` header value directly. The acceptable format description of the 
part must be properly quoted if necessary to not interfere with the root `Accept`
header description.

The following example shows the usage of format description for parts of the 
coverage. Note that not all parts are described, those parts are encoded in the 
default format. As can be seen, multiple formats can be specified to allow for 
encoding negotiation.
```html
Accept: multipart/related; domainset="application/json,text/xml" rangetype=text/xml; rangeset="image/tiff;interleave=Pixel"
```

The resulting multipart document must provide the overall `Content-Type` of `multipart/mixed` and the negotiated `Content-Type` for each part.

E.g:

```html
server: gunicorn/19.9.0
date: Tue, 18 Jun 2019 14:14:14 GMT
connection: close
content-length: 496897
content-type: multipart/related; boundary=9db765ef886544b8928b0a9902dca366
x-frame-options: SAMEORIGIN

--9db765ef886544b8928b0a9902dca366
Content-Type: text/xml
Content-Length: 6354

...

--9db765ef886544b8928b0a9902dca366
Content-Type: text/xml
Content-Length: 1253

...

--9db765ef886544b8928b0a9902dca366
Content-Type: text/xml
Content-Length: 1993

...

--9db765ef886544b8928b0a9902dca366
Content-Type: image/tiff
Content-Disposition: attachment; filename="mosaic_MER_FRS_1PNPDE20060816_090929_000001972050_00222_23322_0058_RGB_reduced_20190618141413.tif"
Content-Length: 486768

...

```

## Open Issues

* Establish service parameter access, based on [OGC API - Common](https://github.com/opengeospatial/oapi_common)
* What is the output format of items typically returned as XML or JSON, such as DomainSet and RangeType? Should maybe FORMAT be applicable here as well? If so, should it be listed as a possible output format (which might be confusing)?
* [OGC 14-121 Web Query Service](https://github.com/opengeospatial/ogc_api_coverages/blob/master/CIS%2BWCS-standards/14-121_Web-Query-Service_2016-06-19.pdf) provides a definition of path syntax, but adds more functionality (such as selection predicates), all based on the XPath standard. Such extra functionality might come handy.
* Add `bbox` and `time` as defined in OGC API - Common as subsetting option.
