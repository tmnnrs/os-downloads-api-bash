# OS Downloads API

Bash scripts which can be used to request Ordnance Survey datasets from the [OS Downloads API](https://osdatahub.os.uk/docs/downloads/overview) (available through the OS Data Hub).

These are primarily for Linux users – and are intended as an alternative to the *Node.js* script in the [OS Downloads API: Getting started guide](https://osdatahub.os.uk/docs/downloads/gettingStarted).

The scripts make use of the following command-line programs:

- [cURL](https://curl.se/) is a computer software project providing a library and command-line tool for transferring data using various network protocols.
- [jq](https://stedolan.github.io/jq/) is a lightweight and flexible command-line JSON processor.
- [wget](https://www.gnu.org/software/wget/) is a computer program that retrieves content from web servers.
- [md5sum](https://man7.org/linux/man-pages/man1/md5sum.1.html) is a computer program designed to verify data integrity using MD5 checksums.

## OS OpenData downloads

```sh
#!/bin/bash

# productId={productId}
# area={area}
# format={format}
# subformat*={subformat}

for element in "${area[@]}"
do
  url="https://api.os.uk/downloads/v1/products/$productId/downloads?area=$element&format=$format"
  if [[ -n $subformat ]]; then url="$url&subformat=$subformat"; fi

  result=$(curl -s -X GET "$url" -H "accept: application/json")

  fileName=($(echo $result | jq -r '.[].fileName'))
  url=($(echo $result | jq -r '.[].url'))
  md5=($(echo $result | jq -r '.[].md5'))

  for index in "${!url[@]}"
  do
    wget "${url[index]}" -O ${fileName[index]}

    md5_=($(md5sum ${fileName[index]}))

    matchStr="# MD5 checksum (Match) => ${md5[index]} ${md5_[0]}"
    nomatchStr="# MD5 checksum (No match) => ${md5[index]} ${md5_[0]}"

    (shopt -s nocasematch; if [[ "${md5[index]}" == "${md5_[0]}" ]]; then echo $matchStr; else echo $nomatchStr; fi)
  done
done
```

The basic premise is as follows:
- Loop through each of the values in the `area` array. For each value use **cURL** to return the JSON metadata for the specified OS OpenData product.
- Use **jq** to extract an array of the `url` strings from the JSON response.
- Loop through each of the values in the `urls` array. Use **wget** to download the listed ZIP file [redirect].
- Validate that the ZIP file has downloaded correctly using the **md5sum** command.

The following input variables are used:

Name | Type | Description
--- | --- | ---
`productId` | *string* | The id of the product.
`area` | *array* | Filter the list of downloads to only include those which cover this area. 'GB' is all of Great Britain, while codes like 'HP' cover National Grid Reference squares.
`format` | *string* | Filter the list of downloads to only include those with this format.
`subformat` | *string* | [Optional] Filter the list of downloads to only include those with this subformat.

NOTE: The `format` and `subformat` values should be URL encoded (which converts characters into a format that can be transmitted over the Internet).

The following table provides a listing of the valid URL encoded `format` values:

Plain | Encoded
--- | ---
`ASCII Grid and GML (Grid)` | `ASCII+Grid+and+GML+%28Grid%29`
`CSV` | `CSV`
`DXF` | `DXF`
`ESRI® Shapefile` | `ESRI%C2%AE+Shapefile`
`GeoPackage` | `GeoPackage`
`GeoTIFF` | `GeoTIFF`
`GML` | `GML`
`MapInfo® TAB` | `MapInfo%C2%AE+TAB`
`TIFF-LZW` | `TIFF-LZW`
`Vector Tiles` | `Vector+Tiles`
`Zip file (containing EPS, Illustrator and TIFF-LZW)` | `Zip+file+%28containing+EPS%2C+Illustrator+and+TIFF-LZW%29`

Alternatively – **GET** `/products` can be used to return a list of the products that are available to download: https://api.os.uk/downloads/v1/products

## Premium data downloads

```sh
#!/bin/bash

# apiKey={apiKey}
# dataPackageId={dataPackageId}
# versionId={versionId}|latest

url="https://api.os.uk/downloads/v1/dataPackages/$dataPackageId/versions/$versionId"

result=$(curl -X GET "$url" -H "accept: application/json" -H "key: $apiKey")

# sudo apt-get install jq
fileName=($(echo $result | jq -r '.downloads[].fileName'))
url=($(echo $result | jq -r '.downloads[].url'))
md5=($(echo $result | jq -r '.downloads[].md5'))

for index in "${!url[@]}"
do
  wget "${url[index]}" -O ${fileName[index]} --header="key: $apiKey"

  md5_=($(md5sum ${fileName[index]}))

  matchStr="# MD5 checksum (Match) => ${md5[index]} ${md5_[0]}"
  nomatchStr="# MD5 checksum (No match) => ${md5[index]} ${md5_[0]}"

  (shopt -s nocasematch; if [[ "${md5[index]}" == "${md5_[0]}" ]]; then echo $matchStr; else echo $nomatchStr; fi)
done
```

The basic premise is as follows:
- Use **cURL** to return the JSON metadata in relation to a specific data package version (including details of the files that are available to download).
- Use **jq** to extract an array of the `url` strings from the JSON response.
- Loop through each of the values in the `urls` array. Use **wget** to download the listed ZIP file.
- Validate that the ZIP file has downloaded correctly using the **md5sum** command.

The following input variables are used:

Name | Type | Description
--- | --- | ---
`apiKey` | *string* | Valid API key with a Premium or Public Sector Plan.
`dataPackageId` | *string* | Data package ID (as created in the OS Data Hub for the required product).
`versionId` | *string* | Version ID. The 'latest' keyword can also be specified to return the most recent data package version.

Alternatively – **GET** `/dataPackages` can be used to return a list of the data packages that are available to download: https://api.os.uk/downloads/v1/dataPackages
