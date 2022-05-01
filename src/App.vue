<template>
	<v-app id="app">
		<button @click="lol()">
			sdsdsds
		</button>
		<img
			id="image"
			src="./assets/images/qr.png"
			width="150px"
		>
		<!-- <div class="app-content">
			<router-view />
		</div>
		<notifications
			position="top right"
			:width="$vuetify.breakpoint.mobile ? '100%' : '400px'"
		/> -->
	</v-app>
</template>

<script>
export default {
	name: 'App',
	created () {
		// Step 1: Scan for a device with 0xffe5 service
	},
	methods: {
		lol () {
			// navigator.bluetooth.requestDevice({
			// 	filters: [
			// 		{
			// 			services: [0xFEE7]
			// 		}
			// 	],
			// 	// optionalServices: ['49535343-fe7d-4ae5-8fa9-9fafd205e455']
			// 	optionalServices: [0x180A, 0xFF00]
			// })
			// 	.then(function(device) {
			// 		// Step 2: Connect to it
			// 		console.log(device)
			// 		return device.gatt.connect();
			// 	})
			// 	.then(function(server) {
			// 		console.log('server', server)
			// 		// Step 3: Get the Service
			// 		return server.getPrimaryService(0xFF00);
			// 	})
			// 	.then(function(service) {
			// 		console.log('service', service)
			// 		// Step 4: get the Characteristic
			// 		return service.getCharacteristic(0xFF02);
			// 	})
			// 	.then(function(characteristic) {
			// 		console.log('char', characteristic)
			// 		// Step 5: Write to the characteristic
			// 		// var data = new Uint8Array([0xbb, 0x25, 0x05, 0x44]);
			// 		let encoder = new TextEncoder('utf-8');
			// 		let sendMsg = encoder.encode("“TM-U200B $20.00”;CHR$(&HD); CR");
			// 		return characteristic.writeValue(sendMsg);
			// 	})
			// 	.catch(function(error) {
			// 		// And of course: error handling!
			// 		console.error('Connection failed!', error);
			// 	});

        let dialog = document.querySelector('#dialog');
        let printCharacteristic;
        let index = 0;
        let data;

        let image = document.querySelector('#image');
        // Use the canvas to get image data
        let canvas = document.createElement('canvas');
        // Canvas dimensions need to be a multiple of 40 for this printer
        canvas.width = 200;
        canvas.height = 200;
        let context = canvas.getContext('2d');
        // context.drawImage(image, 0, 0, 200, 200, 50, 50, 200, 200);
        context.drawImage(image, 0, 0, 150, 150);
		context.font = '18px serif';
		context.fillStyle = '#000000';
		context.fillText('Hello world', 5, 120);
		document.body.appendChild(canvas)

        let imageData = context.getImageData(0, 0, canvas.width, canvas.height).data;

        function getDarkPixel(x, y) {
          // Return the pixels that will be printed black
          let red = imageData[((canvas.width * y) + x) * 4];
          let green = imageData[((canvas.width * y) + x) * 4 + 1];
          let blue = imageData[((canvas.width * y) + x) * 4 + 2];
        //   red, green, blue
          return (red + green + blue) > 0 ? 1 : 0;
        }

        function getImagePrintData() {
          if (imageData == null) {
            console.log('No image to print!');
            return new Uint8Array([]);
          }
          // Each 8 pixels in a row is represented by a byte
          let printData = new Uint8Array(canvas.width / 8 * canvas.height + 8);
          let offset = 0;
          // Set the header bytes for printing the image
          printData[0] = 29;  // Print raster bitmap
          printData[1] = 118; // Print raster bitmap
          printData[2] = 48; // Print raster bitmap
          printData[3] = 0;  // Normal 203.2 DPI
          printData[4] = canvas.width / 8; // Number of horizontal data bits (LSB)
          printData[5] = 0; // Number of horizontal data bits (MSB)
          printData[6] = canvas.height % 256; // Number of vertical data bits (LSB)
          printData[7] = canvas.height / 256;  // Number of vertical data bits (MSB)
          offset = 7;
          // Loop through image rows in bytes
          for (let i = 0; i < canvas.height; ++i) {
            for (let k = 0; k < canvas.width / 8; ++k) {
              let k8 = k * 8;
              //  Pixel to bit position mapping
              printData[++offset] = getDarkPixel(k8 + 0, i) * 128 + getDarkPixel(k8 + 1, i) * 64 +
                          getDarkPixel(k8 + 2, i) * 32 + getDarkPixel(k8 + 3, i) * 16 +
                          getDarkPixel(k8 + 4, i) * 8 + getDarkPixel(k8 + 5, i) * 4 +
                          getDarkPixel(k8 + 6, i) * 2 + getDarkPixel(k8 + 7, i);
            }
          }
          console.log(printData)
          return printData;
        }

        function handleError(error) {
          console.log(error);
          printCharacteristic = null;
          dialog.open();
        }

        function sendNextImageDataBatch(resolve, reject) {
          // Can only write 512 bytes at a time to the characteristic
          // Need to send the image data in 512 byte batches
          if (index + 512 < data.length) {
            printCharacteristic.writeValue(data.slice(index, index + 512)).then(() => {
              index += 512;
              sendNextImageDataBatch(resolve, reject);
            })
            .catch(error => reject(error));
          } else {
            // Send the last bytes
            if (index < data.length) {
              printCharacteristic.writeValue(data.slice(index, data.length)).then(() => {
                resolve();
              })
              .catch(error => reject(error));
            } else {
              resolve();
            }
          }
        }

        function sendImageData() {
          index = 0;
          data = getImagePrintData();
          console.log(data);
        //   getImagePrintData();
          return new Promise(function(resolve, reject) {
            sendNextImageDataBatch(resolve, reject);
          });
        }

        function sendTextData() {
          // Get the bytes for the text
          let encoder = new TextEncoder("utf-8");
          // Add line feed + carriage return chars to text
          let text = encoder.encode('sd' + '\u000A\u000D');
          return printCharacteristic.writeValue(text).then(() => {
            console.log('Write done.');
          });
        }

		function sendPrinterData() {
			// Print an image followed by the text
			sendImageData()
				.then(sendTextData)
				.then(() => {
				})
				.catch(handleError);
		}

		if (printCharacteristic == null) {
			navigator.bluetooth.requestDevice({
				filters: [{
					services: [0xFEE7]
				}],
				optionalServices: [0x180A, 0xFF00]
			})
			.then(device => {
				console.log('> Found ' + device.name);
				console.log('Connecting to GATT Server...');
				return device.gatt.connect();
			})
			.then(server => server.getPrimaryService(0xFF00))
			.then(service => service.getCharacteristic(0xFF02))
			.then(characteristic => {
				// Cache the characteristic
				printCharacteristic = characteristic;
				sendPrinterData();
			})
			.catch(handleError);
		} else {
			sendPrinterData();
		}
	}
	}
}
</script>

<style lang="scss" scoped>
#app {
	background-color: #d4e5ea;
}
.app-content {
	max-width: 400px;
	overflow: hidden;
	width: 100%;
	height: 100%;
	margin: 0 auto;
	background-color: #fff;
}
</style>
