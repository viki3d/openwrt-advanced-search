# openwrt-advanced-search
### <a href="http://openwrt.viki3d.com/demo.html" target="_blank">DEMO</a>

![openwrt-advanced-search-02.png](/openwrt-advanced-search-02.png?id=1 "openwrt-advanced-search main schema")

### CONTENTS
## <a href="#c1"      >1. Download router's database</a>  
### <a href="#c1_1"   >1.1. HTTP request</a>  
#### <a href="#c1_1_1">1.1.1 Raw</a>  
#### <a href="#c1_1_2">1.1.2 CURL</a>  
#### <a href="#c1_1_3">1.1.3 PHP</a>  
## <a href="#c2"      >2. Building the page</a>  
### <a href="#c2_1"   >2.1. Bootstrap 5</a>  
#### <a href="#c2_1"   >2.1.1. Imports</a>  
#### <a href="#c2_1"   >2.1.2. Tooltips</a>  
### <a href="#c2_2"    >2.2. Javascript</a>  
### <a href="#c2_3"    >2.3. Javascript</a>  
### <a href="#c2_4"    >2.4. Angular</a>  

## <span id="c1">1. Download router's database</span>
### <span id="c1_1">1.1. HTTP requests</span>
#### <span id="c1_1_1">1.1.1. Raw</span>
On first call we are downlading the file <b><i>'toh_dump_tab_separated.gz'</i></b>
```
GET /_media/toh_dump_tab_separated.gz HTTP/2
Host: openwrt.org
```

```
HTTP/2 200
Server: nginx
Date: Wed, 28 Nov 2022 08:51:02 GMT
Content-Type: application/octet-stream
Content-Length: 335447
Pragma: no-cache
Cache-Control: public, proxy-revalidate, no-transform, max-age=68400
Last-Modified: Wed, 28 Nov 2022 05:10:26 GMT
ETag: "620379bb9bd27e4fc58155fe4ab5a269"
Content-Disposition: attachment; filename="toh_dump_tab_separated.gz";
Accept-Ranges: bytes
Strict-Transport-Security: max-age=31536000
```
After completing this first step we are obtaining <b><i>'Etag'</i></b> of the requested resource.  
After having the <b><i>'Etag'</i></b> we can include it into the next request as <b><i>'If-None-Match'</i></b>:  

```
GET /_media/toh_dump_tab_separated.gz HTTP/2
Host: openwrt.org
If-None-Match: "620379bb9bd27e4fc58155fe4ab5a269"
```

If the file on server HAS been changed - it will be downloaded again.
If the file on server HAS NOT been changed - the server will respond with <i>HTTP 304</i>. This
means the file <i>will not be downloaded again</i>.

```
HTTP/2 304
Server: nginx
Date: Wed, 28 Nov 2022 09:13:28 GMT
Pragma: no-cache
Expires: Thu, 01 Dec 2022 04:13:28 GMT
Cache-control: public, proxy-revalidate, no-transform, max-age=68400
Last-modified: Wed, 28 Nov 2022 05:10:26 GMT
Etag: "620379bb9bd27e4fc58155fe4ab5a269"
Strict-Transport-Security: max-age=31536000
```

Another way to check if file has been changed is to fire HEAD HTTP request:

```
HEAD /_media/toh_dump_tab_separated.gz HTTP/2
Host: openwrt.org
```

The server will repond with <i>headers only</i>. We can now check the returned <i>Etag</i> value and compare to our's.

#### <span id="c1_1_2">1.1.2. CURL</span>
Test the request using the <i>curl</i> tool:
```
curl https://openwrt.org/_media/toh_dump_tab_separated.gz \
  -X GET  \
  -H 'If-None-Match: "620379bb9bd27e4fc58155fe4ab5a269"' \
  -o toh.gz \
  -D responseheaders.txt
```

#### <span id="c1_1_3">1.1.3. PHP</span>
Get the <b><i>Etag</i></b> with HEAD request:
```
<?php

	const TOH_FILE_TO_DOWNLOAD_ZIP = "https://openwrt.org/_media/toh_dump_tab_separated.zip";
	
	$etag = getEtagWithHeadRequest(TOH_FILE_TO_DOWNLOAD);

	function getEtagWithHeadRequest($url) {
		$ch = curl_init();
		// Set the connection url
		curl_setopt($ch, CURLOPT_URL, $url);
		// Set HTTP request type = HEAD 
		curl_setopt($ch, CURLOPT_NOBODY, true);
		// Include headers into response
		curl_setopt($ch, CURLOPT_HEADER, true);
		// curl_exec return response as a result, instead writing it into console
		curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
		// Do not verify host's SSL certificate
		curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false); 

		$responseHeaders = curl_exec($ch);

		curl_close($ch);

		// Convert to array of rows:
		$headersArr = explode(PHP_EOL, $responseHeaders);
		
		$etag = "";
		foreach ($headersArr as $header) {
			if ( strtoupper(substr($header, 0, strlen("ETAG"))) == "ETAG") {
				// Get everything after ':'
				$val = trim( substr($header, strpos($header, ":")+1, strlen($header))  );
				// Remove quotes
				$etag = substr($val, 1, strlen($val)-2 );
			}
		}	
		
		if ($etag == "") {
			error_log("ETAG_NOT_FOUND");
			return "ETAG_NOT_FOUND";
		}
		return $etag;
	}
?>
```
Unzip .gz file:
```
<?php

	function unGzFile($inpFilename, $outFilename) {
		// Raising this value may increase performance
		$buffer_size = 4096; // read 4kb at a time

		// Open our files (b is for binary mode)
		$inp = gzopen($inpFilename, 'rb');
		$out = fopen($outFilename, 'w');

		// Keep repeating until the end of the input file
		while(!gzeof($inp)) {
			// Read buffer-size bytes
			// Both fwrite and gzread and binary-safe
			fwrite($out, gzread($inp, $buffer_size));
		}

		// Files are done, close files
		fclose($out);
		gzclose($inp);
	}

?>
```
Unzip .zip file  
The file "<i>toh_dump_tab_separated.zip</i>" is not usual .zip archive but an SFX-ZIP archive, which is not readable from the PHP ZipArchive package. And we do not want to use custom implementation or 3rd party library for that. That's why we go for the .gz version of the file.
```
<?php
	function unZipFile($inpFile, $outFolder) {
		$zip = new ZipArchive;
		$res = $zip->open($inpFile);
		if ($res === TRUE) {
			$zip->extractTo($outFolder);
			$zip->close();
		} 
		else {
			switch($res){
				case ZipArchive::ER_EXISTS:
					$ErrMsg = "(". ZipArchive::ER_EXISTS . ") File already exists.";
					break;
				case ZipArchive::ER_INCONS:
					$ErrMsg = "(". ZipArchive::ER_INCONS . ") Zip archive inconsistent.";
					break;
				case ZipArchive::ER_MEMORY:
					$ErrMsg = "(". ZipArchive::ER_MEMORY . ") Malloc failure.";
					break;
				case ZipArchive::ER_NOENT:
					$ErrMsg = "(". ZipArchive::ER_NOENT . ") No such file.";
					break;
				case ZipArchive::ER_NOZIP:
					$ErrMsg = "(". ZipArchive::ER_NOZIP . ") Not a zip archive.";
					break;
				case ZipArchive::ER_OPEN:
					$ErrMsg = "(". ZipArchive::ER_OPEN . ") Can't open file.";
					break;
				case ZipArchive::ER_READ:
					$ErrMsg = "(". ZipArchive::ER_READ . ") Read error.";
					break;
				case ZipArchive::ER_SEEK:
					$ErrMsg = "(". ZipArchive::ER_SEEK . ") Seek error.";
					break;
				default:
					$ErrMsg = "Unknow (Code $rOpen)";
					break;
			}
			error_log('ZipArchive Error: ' . $ErrMsg);
			die('ZipArchive Error: ' . $ErrMsg);
		}
	}
?>
```

Bootstrap 5 depends from 3rd party dependency: Popper, which should be available by including <i>bootstrap.bundle.min.js</i>. However, we do not want to import all that stuff but only the minimal Bootstrap: <i>bootstrap.min.js</i>, which involves the need of explicit import of the Popper dependency:
```
<head>
	...
	<!-- Popper -->
	<script src="./plugins/popper/2.11.6/popper.min.js"></script>
	<!-- 
	<script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/2.11.6/umd/popper.min.js"></script>
	-->
	
	<!-- Bootstrap (@depends jQuery or Popper) -->
	<link href="./plugins/bootstrap/5.0.2/css/bootstrap.min.css" rel="stylesheet" />
	<link href="./plugins/bootstrap/5.0.2/css/bootstrap-grid.min.css" rel="stylesheet" />
	<link href="./plugins/bootstrap/5.0.2/css/bootstrap-utilities.min.css" rel="stylesheet" />
	<script src="./plugins/bootstrap/5.0.2/js/bootstrap.min.js"></script>
```


To enable Bootstrap 5 tooltips we need a pinch of Javascript:
```
// ----------------------------------------------------------------
// Initialize (enable) tool-tips on this page (uses popper.js)
// ----------------------------------------------------------------
var tooltipTriggerList = [].slice.call(document.querySelectorAll('[data-bs-toggle="tooltip"]'));
var tooltipList = tooltipTriggerList.map(function (tooltipTriggerEl) {
	let tootlip = new bootstrap.Tooltip(tooltipTriggerEl);
	// Hide tooltip after the host element has been clicked:
	tooltipTriggerEl.addEventListener('click', function() {
		tootlip.hide();
	});
	return tootlip;
});
```


