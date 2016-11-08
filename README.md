# HAPI Specification 
Version 1.0 | Heliophysics Data and Model Consortium (HDMC) | October 11, 2016 

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Introduction](#introduction)
- [Endpoints](#endpoints)
  - [hapi](#hapi)
    - [Response](#response)
  - [capabilities](#capabilities)
    - [Response](#response-1)
  - [catalog](#catalog)
  - [info](#info)
    - [Response](#response-2)
  - [data](#data)
    - [Response](#response-3)
    - [Data Stream Content](#data-stream-content)
- [HTTP Status Codes](#http-status-codes)
  - [HAPI Client Error Handling](#hapi-client-error-handling)
- [Representation of Time](#representation-of-time)
- [Additional Keyword / Value Pairs](#additional-keyword--value-pairs)
- [More About](#more-about)
  - [Data Types](#data-types)
  - [The ‘size’ Attribute](#the-size-attribute)
  - ['fill' Values](#fill-values)
  - [Data Streams](#data-streams)
- [Security Notes](#security-notes)
- [References](#references)
- [Contact](#contact)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Introduction

This document describes the Heliophysics Application Programmer’s Interface (HAPI) specification for a set of services to enable access to digital Heliophysics time series data. The focus of HAPI is to provide a uniform way for large or small data providers to expose one or more data holdings in an interoperable way. The interface methods required to be a HAPI-compliant server comprise a minimum but complete set of capabilities needed to provide access to digital time series data, with a particular focus on uniform access to digital content, and less of a focus on data discovery. The HAPI specification is intended to be simple enough to be added as an additional access pathway by an existing, large-scale data provider. For small data providers who are currently exposing data only as files in an FTP or HTTP site, the HAPI specification is simple enough that it could be implemented as a primary access mechanism, allowing small provider data holdings to participate in a larger data environment. A uniform access layer to many data holdings can then become a foundation for building more interoperable data services.

The API is built using REST principles that emphasize URLs as stable endpoints through which clients can request data. Because it is based on well-established HTTP request and response rules, a wide range of HTTP clients can be used to interact with HAPI servers.

These definitions are provided first to ensure clarity in ensuing descriptions.

**parameter** – a measured science quantity or a related ancillary quantity; may be a scalar or may have one or  more dimensions; can have units and can have a fill value that represents no measurement or absent information

**dataset** – a collection with a conceptually uniform of set of parameters; a dataset is a logical entity, but it often physically resides in one or more files that are accessible online; a HAPI service presents a dataset as a seamless collection of records, offering a way to retrieve the ordered parameters while hiding actual storage details

**request parameter** – keywords that appear after the ‘?’ in a URL with a GET request.

In this example GET request:
```
     http://example.com/data?id=alpha&time.min=2016-07-13 
```
the two request parameters, id and time.min, are shown in bold and have values of ‘alpha’ and ‘2016-07-13’ respectively. This document will always use the full phrase ‘request parameter’ to refer to these URL elements to draw a clear distinction from a parameter in a dataset.

# Endpoints

The HAPI specification consists of four required endpoints that give clients a precise way to determine the data holdings of the server and to request data from the server.

1. describe the capabilities of the server; this lists the output formats the server can emit (CSV and binary)
2. list the datasets that are available; each dataset is associated with a unique id
3. obtain a description for dataset of a given id; the description defines the parameters in every dataset record
4. stream data content for a dataset of a given id; the streaming request must have time bounds (specified by request parameters time.min and time.max) and may indicate a subset of parameters (default is all parameters)

There is also an optional landing page endpoint that returns human-readable HTML, and although there is recommended content for this landing page, it is not essential to the functioning of the server.

The four required endpoints behave like REST-style services, in that the resulting HTTP response is the complete response for each endpoint. The responses do not contain links or URLS pointing to data, rather the response stream contains the requested data. The specification for each endpoint is discussed below.

All endpoints must be directly below a hapi path element in the URL:
```
http://example.com/hapi/catalog
http://example.com/hapi/capabilities
http://example.com/hapi/info
http://example.com/hapi/data
```
The input specification for each endpoint (the request parameters and their allowed values) must be strictly enforced. Only the request parameters described below are allowed, and no extensions are permitted. If a HAPI server sees a request parameter that it does not recognize, it is required to throw an error indicating that the request is invalid (via HTTP 400 error – see below).  A server that ignored an unknown request parameter would falsely indicate to clients that the request parameter was understood and was taken into account when creating the output.

All requests to a HAPI server are for retrieving resources and must not change the server state. Therefore, all HAPI endpoints must respond only to HTTP GET requests. POST requests should result in an error. This represents a RESTful approach in which GET requests are restricted to be read-only operations from the server. The HAPI specification does not allow any input to the server (which for RESTful services are often implemented using POST requests). 

The following is the detailed specification for the four main HAPI endpoints.

## hapi

This root endpoint is optional and serves as a human-readable landing page for the server. Unlike the other endpoints, there is no strict definition for the output, but if present, it should include a brief description of the other endpoints, and links to documentation on how to use the server. An example landing page that can be easily customized for a new server is available at the spase-group.org/hapi web site.

**Sample Invocation**
```
http://example.com/hapi
```

**Request Parameters**

None

### Response

The response is in HTML format and is not strictly defined, but should look something like the example below.

**Example**

Retrieve a listing of resources shared by this server.
```
http://example.com/hapi
```
Response:
```
<html>
<head> </head>
<body>
<h2> HAPI Server<h2>
<p> This server supports the HAPI 1.0 specification for delivery of time series
    data. The server consists of the following 4 REST-like endpoints that will
    respond to HTTP GET requests.
</p>
<ol>
<li> <a href="capabilities">capabilities</a> describe the capabilities of the server; this lists the output formats the server can emit (CSV and binary)</li>
<li><a href="catalog">catalog</a> list the datasets that are available; each dataset is associated with a unique id</li>
<li><a href="info">info</a> obtain a description for dataset of a given id; the description defines the parameters in every dataset record</li>
<li><a href="data">data</a> stream data content for a dataset of a given id; the streaming request must have time bounds (specified by request parameters time.min and time.max) and may indicate a subset of parameters (default is all parameters)</li>
</ol>
<p> For more information, see <a href="http://spase-group.org/hapi">this HAPI description</a> at the SPASE web site.  </p>
</body>
<html>
```

## capabilities

This endpoint describes relevant implementation capabilities for this server. Currently, the only possible variability from server to server is the list of output formats that are supported. 

A server must support CSV output format, but binary output format is optional.

**Sample Invocation**
```
http://example.com/hapi/capabilities
```

**Request Parameters**

None

### Response

Response is in JSON format [3] as defined by RFC-7159 and has a mime type of “application/json”. The format of the JSON response is defined in the section “HAPI Metadata.” The capabilities object has various keyword value pairs indicating the implemented features for a given server capability. Only one capability is currently in the spec, the “formats” capability, and it must be present in the response.

**Capabilities**

| Name     | Type     | Description |
| -------- | -------- | ----------- |
| HAPI     | string   | **Required**<br/>The version number of the HAPI specification this description complies with. |
| capabilities | array(endpoint) | **Required**<br/>A list of capabilities offered by his server. |

**Capabilities Object**

| Name     | Type      | Description |
| -------- | --------- | ----------- |
| formats  | string array | **Required**<br/> The list of output formats the serve can emit. The allowed values in the last are “csv” and “binary”. All HAPI servers must support at least “csv” output format, but “binary” output format is optional. |

**Example**

Retrieve a listing of resources shared by this server.
```
http://example.com/hapi/capabilities
```
Response:
```
{
  "HAPI": "1.0",
  "capabilities": [
    {
      "formats": [ "csv", "binary" ]
    }
  ]
}
```
If a server only reports an output format of CSV, then requesting data in binary form should cause the server to issue an HTTP return code of 400 (bad request).

Servers may support their own custom output formats, which would be advertised here.

## catalog

Provides a list of datasets available via this server.

**Sample Invocation**
```
http://example.com/hapi/catalog
```

**Request Parameters**

None

### Response

The response is in JSON format [3] as defined by RFC-7159 and has a mime type of “application/json”. An example is given here, and the content of the JSON response is fully defined in the section “HAPI JSON Content.” The catalog is a simple listing of identifiers for the datasets available through the server providing the catalog. Additional metadata about each dataset is available through the info endpoint (described below). The catalog takes no query parameters and always lists the full catalog.

**Catalog**

| Name   | Type    | Description |
| ------ | ------- | ----------- |
| HAPI   | string  | **Required**<br/> The version number of the HAPI specification this description complies with. |
| catalog | array(endpoint) | **Required**<br/>A list of endpoints available from this server. |

**Catalog Object**

| Name   | Type    | Description |
| ------ | ------- | ----------- |
| id     | string  | **Required**<br/> The identifier that the host system uses to locate the resource. If the "id" is a URL it should be considered an reference to a HAPI service on another server. |

**Example**

Retrieve a listing of resources shared by this server.
```
http://example.com/hapi/catalog
```
Response:
```
{
   "HAPI" : "1.0",
   "catalog" : 
   [
      {"id": "path/to/ACE_MAG"},
      {"id": "data/IBEX/ENA/AVG5MIN"},
      {"id": "data/CRUISE/PLS"},
      {"id": "any_identifier_here"}
   ]
}
```
Dataset identifiers in the catalog should be stable over time. Including version numbers or other revolving elements (dates, processing ids, etc.) in the datasets identifiers is not desirable. The intent of the HAPI specification is to allow data to be referenced using RESTful URLs that have a reasonable lifetime.

Also, note that the identifiers can have slashes in them. The identifiers must be unique within a single HAPI server.

## info

Lists a data header for a given dataset, including a description of the parameters in the dataset.

**Sample Invocation**
```
http://example.com/hapi/info?id=ACE_MAG
```

**Request Parameters**

| Name       | Description |
| ---------- | ----------- |
| id         | **Required**<br/> The identifier for the resource. |
| parameters | **Optional**<br/> A subset of the parameters to include in the header. |

### Response

The response is in JSON format [3] and provides metadata about one dataset. The focus is a list of parameters in the dataset, although there are several required and many optional descriptive keywords that can be included. Custom, user-defined keywords may also be present, but they must conform to the name specifications described in the section about.

The first parameter in the data must be a time column (type of "isotime") and this must be the independent variable for the dataset.


**Info**

| Dataset Attribute | Type    | Description |
| ----------------- | ------- | ----------- |
| HAPI              | string  | **Required**<br/> The version number of the HAPI specification this description complies with.|
| format            | string  | **Required** when header is prefixed to data stream.<br/> Format of the data as "csv" or "binary". |
| parameters        | array(Parameter) | **Required**<br/> Description of the parameters in the data |
| firstDate         | string  | **Optional**<br/> ISO 8601 date of first record of data |
| lastDate          | string  | **Optional**<br/> ISO 8601 date for the last record of data |
| sampleStartDate   | string  | **Optional**<br/> The end time of a sample time period for a dataset, where the time period must contain a manageable, representative example of valid, non-fill data. |
| sampleEndDate     | string  | **Optional**<br/> The end time of a sample time period for a dataset, where the time period must contain a manageable, representative example of valid, non-fill data |
| description       | string  | **Optional**<br/> A brief description of the resource |
| resourceURL       | string  | **Optional**<br/> URL linking to more detailed information about this data |
| resourceID        | string  | **Optional**<br/> An identifier by which this data is known in another setting, for example, the SPASE ID. |
| creationDate      | string  | **Optional**<br/> ISO 8601 Date and Time of the dataset creation. |
| modificationDate  | string  | **Optional**<br/> Last modification time as an ISO 8601 date |
| deltaTime         | string  | **Optional**>br/> Time difference between records as an ISO 8601 duration 
| contact           | string  | **Optional**<br/> Relevant contact person and possibly contact information. |
| contactID         | string  | **Optional**<br/> The identifier in the discovery system for information about the contact. For example, the SPASE ID of the person. |

**Parameter** 

| Parameter Attribute | Type    | Description |
| ------------------- | ------- | ----------- |
| name                | string  | **Required**
| type                | string  | **Required**<br/> One of "string", "double", "integer", “isotime". Content for "double" is always 8 bytes in IEEE-754 format, "integer" is 4 bytes little-endian.  There is no default length for ‘string’ and ‘isotime’ types. |
| length              | integer | **Required** for type ‘string’ and ‘isotime’; **not allowed for others**<br/> The number of bytes or characters that contain the value. Valid only if data is streamed in binary format. |
| units               | string  | **Optional<br/> The units for the data values represented by this parameter. Default is ‘dimensionless’ for everything but ‘isotime’ types.
| size                | array of integers | **Required** for array parameters; **not allowed for others**<br/> The number of elements in the 'size' array indicates how many dimensions the parameter has, and the value of each element indicates the length of each dimension. Thus '[7]' indicates a 1-D array of length 7.  Array parameters are unwound and each column (in the CSV or binary output) is one slice of the array. See below for more about array sizes.  |
| fill                | string  | **Optional**<br/> A fill value indicates no valid data is present.  See below for issues related to specifying fill values as strings. |
| description         | string  | **Optional**<br/> A brief description of the parameter |
| bins                | object  | **Optional**<br/> For array parameters, the bins object describes the values associated with each element in the array. If the parameter represents a frequency spectrum, the bins object captures the frequency values for each frequency bin. The center value for each bin is required and the min and max values are optional. If min or max is present, the other is also required. The bins object has an optional “units” keyword (any string value is allowed) and a required “values” keyword that holds a list of objects containing the “min”, “center”, and “max” for each bin. See below for an example showing a parameter that holds a proton energy spectrum. |

**Example**
```
http://example.com/hapi/info?id=path/to/ACE_MAG
```
Response:
```
{  "HAPI": "1.0",
   "creationDate”: "2016-06-15T12:34"
   "parameters": [
       { "name": "Time",
         "type": "isotime",
         "length": 24 },
       { "name": "radial_position",
         "type": "double",
         "units": "km",
         "description": "radial position of the spacecraft" },
       { "name": "quality flag",
         "type": "integer",
         "units ": "none ",
         "description ": "0=OK and 1=bad " },
       { "name": "mag_GSE",
         "type": "double",
         "units": "nT",
         "size" : [3],
         "description": "hourly average Cartesian magnetic field in nT in GSE" }
   ]
}
```
There is an interaction between the info endpoint and the data endpoint, because the header from the info endpoint describes the record structure of data emitted by the data endpoint. Thus after a single call to the info endpoint, a client could make multiple calls to the data endpoint (for multiple time ranges, for example) with the expectation that each data response would contain records described by the single call to the info endpoint. The data endpoint can optionally prefix the data stream with header information, potentially obviating the need for the info endpoint. But the info endpoint is useful in that allows clients to learn about a dataset without having to make a data request.

Both the info and data endpoints take an optional request parameter (recall the definition of request parameter in the introduction) called ‘parameters’ that allows users to restrict the dataset parameters listed in the header and data stream, respectively. This enables clients (that already have a list of dataset parameters from a previous info or data request) to request a header for a subset of parameters that will match a data stream for the same subset of parameters. Consider the following dataset header for a fictional dataset with the identifier MY_MAG_DATA.
```
{  "HAPI": "1.0",
   "creationDate”: "2016-06-15T12:34"
   "parameters": [
       { "name": "Time",
         "type": "isotime",
         "length": 24 },
       { "name": "Bx", "type": "double", "units": "nT" },
       { "name": "By", "type": "double", "units": "nT" },
       { "name": "Bz", "type": "double", "units": "nT" },
    ]
}
```
An info request like this:
```
http://example.com/hapi/info?id=MU_MAG_DATA&parameters=Bx
```
would result in a header listing only the one dataset parameter: 
```
{  "HAPI": "1.0",
   "creationDate”: "2016-06-15T12:34"
   "parameters": [
       { "name": "Time",
         "type": "isotime",
         "length": 24 },
       { "name": "Bx", "type": "double", "units": "nT" },
    ]
}
```
Note that the time parameter is still included as the first dataset parameter, even if not requested.

Here is a summary of the effect of asking for a subset of dataset parameters:
- do not ask for any specific parameters (i.e., there is no request parameter called ‘parameters’): all columns
- ask for just the time parameter: just the time column
- ask for a single, non-time dataset parameter (like ‘parameters=Bx’): a time column and one data column
 
The data endpoint also takes the 'parameters' option, and so behaves the same way as the info endpoint in terms of which columns are included in the response.

## data

Provides access to a data resource and allows for selecting time ranges and fields to return. Data is returned as a stream in CSV[2] or binary format. The “Data Stream Formats” section describes the stream contents.

The resulting data stream can be thought of as a stream of records, where each record contains one value for each of the dataset parameters. Each data record must contain a data value or a fill value (of the same data type) for each parameter.
 
**Request Parameters**

| Name         | Description |
| ------------ | ----------- |
| id           | **Required**<br/> The identifier for the resource |
| time.min     | **Required**<br/> The smallest value of time to include in the response |
| time.max     | **Required**<br/> The largest value of time to include in the response | 
| parameters   | **Optional**<br/> A comma separated list of parameters to include in the response. Default is all parameters.|
| include      | **Optional**<br/> Has one possible value of "header" to indicate that the info header should precede the data. The header lines will be prefixed with the "#" character.  |
| format       | **Optional**<br/> The desired format for the data stream. Possible values are "csv" and "binary". |

### Response

Response is either in CSV format as defined by RFC-4180 and has a mime type of "text/csv" or in binary format where floating points number are in IEEE 754[5] format and byte order is LSB. Default data format is CSV. See the section on Data Stream Content for more details.

If the header is requested, then each line of the header must begin with a hash (#) character. Otherwise, the contents of the header should be the same as returned from the info endpoint. When a data stream has an attached header, the header must contain an additional "format" attribute to indicate if the content after the header is "csv" or "binary". Note that when a header is included, the data stream is not strictly in CSV format.

The first parameter in the data must be a time column (type of "isotime") and this must be the independent variable for the dataset. If a subset of parameters is requested, the time column is always provided, even if it is not requested.

### Data Stream Content

The two possible output formats are ‘csv’ and binary’. A HAPI server must support CSV, and binary is optional.

In the CSV stream, each record is one line of text, with commas between the values for each dataset parameter. Array parameters are unwound in the sense that each element in the array goes into its own column. Currently, only 1-D arrays are supported, so the ordering of the unwound columns is just the index ordering of the array elements. It is up to the server to decide how much precision to include in the ASCII values for the dataset parameters.

The binary data output is essentially a binary version of the CSV stream. Recall that the dataset header provides type information for each dataset parameter, and this definitively indicates the number of bytes and the byte structure of each parameter, and thus of each binary record in the stream.  All numeric values are little endian, integers are always four byte, and floating point values are always IEEE 754 double precision values.

Dataset parameters of type ‘string’ and ‘isotime’  (which are just strings of ISO 8601 dates) must have in their header a length element. All strings in the binary stream should be null terminated, and so the length element in the header should include the null terminator as part of the length for that string parameter.

**Examples**

Two examples of data requests and responses are given – one with the header and one without.

**Data with Header**

Note that in this request, the header is requested, so the same header from the info request will be prepended to the data, but with a ‘#’ character as a prefix for every header line.
```
http://example.com/hapi/data?id=path/to/ACE_MAG&time.min=2016-01-01&time.max=2016-02-01&include=header
```
Response:
```
#{
#  "HAPI": "1.0",
#   "creationDate”: "2016-06-15T12:34"
#   "format": "csv",
#   "parameters": [
#       { "name": "Time",
#         "type": "isotime",
#         "length": 24
#       },
#       { "name": "radial_position",
#         "type": "double",
#         "units": "km",
#         "description": "radial position of the spacecraft"
#       },
#       { "name": "quality flag",
#         "type": "integer",
#         "units ": "none ",
#         "description ": "0=OK and 1=bad " 
#       },
#       { "name": "mag_GSE",
#         "type": "double",
#         "units": "nT",
#         "size" : [3],
#         "description": "hourly average Cartesian magnetic field in nT in GSE"
#       }
#   ]
#}
Time,radial_position,quality_flag,mag_GSE_0,mag_GSE_1,mag_GSE_2
2016-01-01T00:00:00.000,6.848351,0,0.05,0.08,-50.98
2016-01-01T01:00:00.000,6.890149,0,0.04,0.07,-45.26
		?
		? 
2016-01-01T02:00:00.000,8.142253,0,2.74,0.17,-28.62
```

**Data Only**

The following example is the same, except it lacks the request to include the header.
```
http://example.com/hapi/data?id=path/to/ACE_MAG&time.min=2016-01-01&time.max=2016-02-01
```
**Response**

If the data resource contains a time field, plus 3 data fields the response will be something like:
```
Time,radial_position,quality_flag,mag_GSE_0,mag_GSE_1,mag_GSE_2
2016-01-01T00:00:00.000,6.848351,0,0.05,0.08,-50.98
2016-01-01T01:00:00.000,6.890149,0,0.04,0.07,-45.26
		?
		? 
2016-01-01T02:00:00.000,8.142253,0,2.74,0.17,-28.62
```

# HTTP Status Codes

Error reporting by the server should be done use standard HTTP status codes. In an error response, the HTTP header should include the code and the body of the error message should include relevant details about the error. It is up to clients then how to ingest and/or display this error information. Servers are encouraged but not required to limit their return codes to the following suggested set:

| Status Code | Description |
| ----------- | ----------- |
| 200         | OK – the user request was fully met with no errors |
| 400         | Bad request – something was wrong with the request, such as an unknown request parameter (misspelled or incorrect capitalization perhaps?), an unknown data parameter, an invalid time range, unknown dataset id |
| 500         | Internal error – the server had some internal fault or failure due to an internal software or hardware problem |

## HAPI Client Error Handling

Because servers are not required to limit HTTP return codes to those in the above table, clients should be able to handle the full range of HTTP responses. Also, a HAPI server may not be the only software to interact with a URL-based request from a HAPI server (there may be a load balancer or upstream request routing or caching mechanism in place), and thus it is just good client-side practice to be able to handle any HTTP errors.

# Representation of Time

All data requests to a HAPI server require a time range (via the time.min and time.max request parameters). The time format for these time values is ISO8601. Servers must be able to handle both the day of year form (YYYY-dddThh\:mm\:ss) as well as the year-month-day (YYYY-mm-ddThh\:mm\:ss) form for these time values. Time values are all considered to be relative to GMT, and so no time zone specifications are allowed on the time values. A trailing “Z” character indicating that the times are indeed in GMT is allowed When making a request to a HThe HAPI specification is focused on access to time series data, so understanding how the server understands and emits time values is important. 

When making a request to the server, the time range (time.min and time.max) values are allowed to beOn the request parameter sideThe time format for the request must be a valid time string according to the ISO 8601 standard. Note that servers should be able to parse both the year-month-day (yyyy-mm-ddThh\:mm\:ss.sss) or day-of-year (yyyy-dddThh\:mm\:ss.sss) orientations of the time string. 

Also, in the data values returned, servers are allowed to use either form for ISO 8601 time strings, so clients must be able to transparently handle both flavors. A server should use a consistent time format for a single dataset.

Time values are considered to be relative to GMT. Time zones or time offsets are not allowed.  The trailing “Z” on the time string is allowed, and if not present, the time value is still assumed to be in GMT.

Note that a fill value can be provided for time parameters. If no fill value for time is specified, the string "0001-01-01T00\:00\:00.000" is used as the default.

Servers should strictly obey the time bounds requested by clients. No data values outside the requested time range should be returned.

Note that the ISO-8601 time format allows arbitrary precision on the time values. HAPI servers should therefore also accept time values with high precision. As a practical limit, servers should at least handle time values down to the nanosecond or picosecond level.

# Additional Keyword / Value Pairs

While the HAPI server strictly checks all request parameters (servers must return an error code given any unrecognized request parameter as described earlier), the JSON content output by a HAPI server may contain additional, user-defined metadata elements.  All non-standard metadata keywords must begin with the prefix “x_” to indicate to HAPI clients that these are extensions. Custom clients could make use of the additional keywords, but standard clients would ignore the extensions. By using the standard prefix, the custom keywords will never conflict with any future keywords added to the HAPI standard. 


# More About 
## Data Types

Note that there are only a few supported data types: isotime, string, integer, and double. This is intended to keep the client code simple in terms of dealing with the data stream. However, the spec may be expanded in the future to include other types, such as 4 byte floating point values (which would be called float), or 2 byte integers (which would be called short).

## The ‘size’ Attribute

The 'size' attribute can only be used for an array. Therefore "size" : [0] is not allowed and cannot be used to refer to a scalar parameter. A scalar parameter will simply not have the ‘size’ attribute. Using ‘size’: [1] refers to a 1-D array of length 1.  Currently only 1-D arrays are supported, but higher dimensionality may be supported and would be expected to look like ‘size’ :[3,5] for a 2-D array with length 3 in one dimension and 5 in the other.

## 'fill' Values

Because the fill values for all types must be specified as a string, care must be taken to accurately represent any desired numeric fill value. For integers, string fill values that are small enough will properly parse to the right internal binary representation. For ‘double’ parameters, the fill string must parse to an exact IEEE 754 double representation. The string ‘NaN’ is allowed, in which case ‘csv’ output should contain the string ‘NaN’ for fill values. For double NaN values, the bit pattern for quiet NaN should be used, as opposed to the signaling NaN, which should not be used (see reference [6]). For ‘string’ and ‘isotime’ parameters, the string ‘fill’ value is used at face value.

## Data Streams

The following two examples illustrate two different ways to represent a magnetic field dataset. The first lists a time column and three data columns, Bx, By, and Bz for the Cartesian components. 
```
{
	"HAPI": "1.0",
	"firstDate": "2016-01-01T00:00:00.000",
	"lastDate": "2016-01-31T24:00:00.000",
	"time": "timestamp",
	"parameters": [
		{"name" : "timestamp", "type": "isotime", "units": "none"},
		{"name" : "bx", "type": "double", "units": "nT"},
		{"name" : "by", "type": "double", "units": "nT"},
		{"name" : "bz", "type": "double", "units": "nT"}		
	}
}
```
This example shows a header for the same conceptual data (time and three magnetic field components), but with the three components grouped into a one-dimensional array of size 3.
```
{
	"HAPI": "1.0",
	"firstDate": "2016-01-01T00:00:00.000",
	"lastDate": "2016-01-31T24:00:00.000",
	"time": "timestamp",
	"parameters": [
    { "name" : "timestamp", "type": "isotime", "units": "none" },
    { "name" : "b_field", "type": "double", "units": "nT","size": [3] }
  ]
}
```
These two different representations affect how a subset of parameters could be requested from a server.  The first example, by listing Bx, By, and Bz as separate parameters, allows clients to request individual components: 
```
http://example.com/hapi/data?id=MY_MAG_DATA&time.min=2001&time.max=2010&parameters=Bx
```
This request would just return a time column (always included as the first column) and a Bx column. But in the second example, the components are all inside a single parameter named ‘b_field’ and so a request for this parameter must always return all the components of the parameter. There is no way to request individual elements of an array parameter.

The following example shows a proton energy spectrum and illustrates the use of the ‘bins’ element. Note also that the uncertainty of the values associated with the proton spectrum are a separate variable. There is currently no way in the HAPI spec to explicitly link a variable to its uncertainties.
```
{"HAPI": "1.0",
 "creationDate": "2016-06-15T12:34",

 "parameters": [
   { "name": "Time",
     "type": "isotime",
     "length": 24
   },
   { "name": "qual_flag",
     "type": "int"
   },
   { "name": "maglat",
     "type": "double",
     "units": "degrees",
     "description": "magnetic latitude"
   },
   { "name": "MLT",
     "type": "string",
     "length": 5,
     "units": "hours:minutes",
     "description": "magnetic local time in HH:MM"
   },
   { "name": "proton_spectrum",
     "type": "double",
     "size": [3],
     "units": "particles/(sec ster cm^2 keV)"
     "bins": {
         "units": "keV",
         "values": [
             { "min": 10.1, "center": 15, "max": 20 },
             { "min": 20,   "center": 25, "max": 30.3 },
             { "min": 30.3, "center": 35, "max": 40 }
              ]
          },
   { "name": "proton_spectrum_uncerts",
     "type": "double",
     "size": [3],
     "units": "particles/(sec ster cm^2 keV)"
     "bins": {
         "units": "keV",
         "values": [
             { "min": 10.1, "center": 15, "max": 20 },
             { "min": 20,   "center": 25, "max": 30.3 },
             { "min": 30.3, "center": 35, "max": 40 }
          ]
   }

  ]
}
```


# Security Notes

When the server sees a request parameter that it does not recognize, it should throw an error.

So given this query

```
http://example.com/hapi/data?id=DATA&time.min=T1&time.max=T2&fields=mag_GSE&avg=5s
```
the server should throw an error with a BAD_REQUEST response code to indicate that an invalid request parameter was provided.  

In following general security practices, HAPI servers should carefully screen incoming request parameter names values.  Unknown request parameters and values should not be echoed in the error response. 

# References

[1] ISO 8601:2004, http://dotat.at/tmp/ISO_8601-2004_E.pdf  
[2] CSV format, https://tools.ietf.org/html/rfc4180  
[3]  JSON Format, https://tools.ietf.org/html/rfc7159  
[4] "JSON Schema", http://json-schema.org/  
[5] EEE Computer Society (August 29, 2008). "IEEE Standard for Floating-Point Arithmetic". IEEE. doi:10.1109/IEEESTD.2008.4610935. ISBN 978-0-7381-5753-5. IEEE Std 754-2008  
[6] IEEE Standard 754Floating Point Numbers, http://steve.hollasch.net/cgindex/coding/ieeefloat.html   

# Contact

Todd King (tking@igpp.ucla.edu)  
Jon Vandegriff (jon.vandegriff@jhuapl.edu)  
Robert Weigel (rweigel@gmu.edu)  
Robert Candey (Robert.M.Candey@nasa.gov)  
Aaron Roberts (aaron.roberts@nasa.gov)  
Bernard  Harris (bernard.t.harris@nasa.gov)  
Nand Lal (nand.lal-1@nasa.gov)  
