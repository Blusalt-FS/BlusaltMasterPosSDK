# Blusalt POS SDK

## Installation

## Step 1
```sh
#Create a github.properties file in the root folder of android. Path should be like below: E.g. "mySampleApp/android/github.properties"

# Add the following in github.properties:
USERNAME_GITHUB=MyUsernameOnGithub
TOKEN_GITHUB=TokenFromGithubClassicToken
```

## Step 2: build.gradle (project level)
Add the following to build.gradle(project level) file
```sh


buildscript {
    repositories {
      ...
    }
    dependencies {
      ...
    }
}

allprojects {
    def githubPropertiesFile = rootProject.file("github.properties")
    def githubProperties = new Properties()
    githubProperties.load(new FileInputStream(githubPropertiesFile))

    repositories {
        ...
                
        maven {
          name "GitHubPackages"
          url 'https://maven.pkg.github.com/Blusalt-FS/BlusaltTrenditPosSDK'

          credentials {
            username githubProperties['USERNAME_GITHUB']
              password githubProperties['TOKEN_GITHUB']
            }
       }
    }
}

```


## Step 3: build.gradle (app)
All the following to build.gradle (app level)

```sh

dependencies{
  implementation 'com.blusalt:master-sdk:1.1.1'
}
```


## Step 4: Usage
```
How to process a transaction and report terminal location.
 
```

## Transaction Processing 

# Initialization 
```sh

        PosSdk.Builder(this)
            .setTransactionMode(TransactionMode.API)
            .setTransactionCurrency(TransactionCurrency.NGN)
            .setBuildType(BuildType.EXTERNAL)
            .setEnvironment(Environment.STAGING)
            .build()
            .initialize()


# ConfigTerminal
```sh

        CoroutineScope(Dispatchers.Default + Job()).launch {
            delay(10000)
            val configStatus = PosSdk.configureTerminal()
            if (configStatus == "Successful") {
                Log.d("APP", "Terminal ready")
                checkLocationPermission()
            } else {
                Log.e("APP", "Terminal not ready: $configStatus")
            }
        }


# Process Transaction
```sh

     fun startEMV(amt: Double) {
        PosSdk.uiHelper.showLoading("Please wait...")

        // Start payment
        lifecycleScope.launch {
            PosSdk.startPayment(
                amt,
                TransType.PURCHASE,
                TransactionCurrency.NGN,
                "YOUR API KEY",
                listener = object : PaymentListener {

                    override fun onInsertCard() {
                        showMessage("Insert Card")
                        PosSdk.uiHelper.showLoading("Insert/Tap your card")
                    }

                    override fun onCardDetected() {
                        showMessage("Card Detected")
                        PosSdk.uiHelper.showLoading("Card Found")
                    }

                    override fun onProcessing() {
                        showMessage("Processing...")
                        PosSdk.uiHelper.showLoading("Processing...")
                    }

                    override fun onPinEntry(length: Int) {
                        showMessage("PIN Length $length")
                    }

                    override fun onEmvRequestOnline(tlvData: TerminalInfoProcessor) {
                        showMessage("collect trans Data... ${Gson().toJson(tlvData)}")
                        PosSdk.uiHelper.hideLoading()
                    }

                    override fun onTransactionResult(response: TerminalResponse) {

                        if (response.status) {
                            showMessage("Approved")
                        } else {
                            showMessage("Declined")
                        }

                        PosSdk.uiHelper.showLoading(response.message)
                        Log.d("SDK", "Printer Status ${PosSdk.getPrinterStatus()}")

                            PosSdk.configureReceiptFooter(
                                "SECURED BY BLUSALT",
                                "https://blusalt.net",
                                "clientservices@blusalt.net",
                                "+234 913 586 4288"
                            )

                            if (PosSdk.getPrinterStatus()) {
                                PosSdk.startPrinting(response, true)
                            } else {
                                //Toast message: Please check the printer paper.
                            }

                        PosSdk.uiHelper.hideLoading()
                    }

                    override fun onError(message: String) {
                        showMessage(message)
                        PosSdk.uiHelper.showLoading(message)
                        PosSdk.uiHelper.hideLoading()
                    }
                }
            )
        }
    }

```sh
   fun showMessage(message: String) {
        println("Message: $message")
    }


## NIbbs Geolocation Report
```sh

  private fun startPosGeo() {
        PosSdk.initializeGeoLocation(
            application,
            object : GeoResultCallBack {
                override fun onSuccess(message: String?) {
                    CoroutineScope(Dispatchers.Main).launch {
                        Toast.makeText(applicationContext, message, Toast.LENGTH_SHORT).show()
                    }
                }

                override fun onError(error: String?) {
                    CoroutineScope(Dispatchers.Main).launch {
                        Toast.makeText(applicationContext, error, Toast.LENGTH_SHORT).show()
                    }
                }
            }
        )
        PosSdk.start(applicationContext)
    }

    private fun checkLocationPermission() {
        val foregroundGranted =
            ContextCompat.checkSelfPermission(
                this,
                Manifest.permission.ACCESS_FINE_LOCATION
            ) == PackageManager.PERMISSION_GRANTED &&
                    ContextCompat.checkSelfPermission(
                        this,
                        Manifest.permission.ACCESS_COARSE_LOCATION
                    ) == PackageManager.PERMISSION_GRANTED

        if (!foregroundGranted) {
            ActivityCompat.requestPermissions(
                this,
                arrayOf(
                    Manifest.permission.ACCESS_FINE_LOCATION,
                    Manifest.permission.ACCESS_COARSE_LOCATION
                ),
                LOCATION_REQ
            )
            return
        }

        // Foreground location granted → start immediately
        startPosGeo()
    }

    override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<out String>,
        grantResults: IntArray
    ) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)

        if (requestCode == LOCATION_REQ &&
            grantResults.isNotEmpty() &&
            grantResults.all { it == PackageManager.PERMISSION_GRANTED }
        ) {
            // Permission JUST granted → start immediately
            startPosGeo()
        } else {
            Toast.makeText(
                this,
                "Permissions required to start Blusalt Service",
                Toast.LENGTH_LONG
            ).show()
        }
    }

    companion object {
        private const val LOCATION_REQ = 1001
    }

