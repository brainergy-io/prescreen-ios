# BrainergyPrescreen

[![CI Status](https://img.shields.io/travis/northanapon/BrainergyPrescreen.svg?style=flat)](https://travis-ci.org/northanapon/BrainergyPrescreen)
[![Version](https://img.shields.io/cocoapods/v/BrainergyPrescreen.svg?style=flat)](https://cocoapods.org/pods/BrainergyPrescreen)
[![License](https://img.shields.io/cocoapods/l/BrainergyPrescreen.svg?style=flat)](https://cocoapods.org/pods/BrainergyPrescreen)
[![Platform](https://img.shields.io/cocoapods/p/BrainergyPrescreen.svg?style=flat)](https://cocoapods.org/pods/BrainergyPrescreen)

## Example

To run the example project, clone the repo, and run `pod install` from the Example directory first.

### How-to

- Initialize Prescreen instance

  ```swift
  Prescreen.shareInstance.initialize(apiKey: "API_KEY")
  ```

- Use `AVCaptureVideoDataOutputSampleBufferDelegate` to capture the images from a camera and pass the camera output to the `scanIDCardSync` function:

  ```swift
    extension ViewController: AVCaptureVideoDataOutputSampleBufferDelegate {
        func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
            let result = Prescreen.shareInstance.scanIDCardSync(sampleBuffer: sampleBuffer, cameraPosition: self.currentDevice.position)
            if result.error != nil {
                // handle error
            }
            else if (result.confidence >= 0.5) {
                // handle successful screening
            }
        }
    }

  ```

- The `result` is an object of `IDCardResult` class and consists of the following fields:

  ```swift
  public struct IDCardResult {
    public var error: Error?
    public var confidence: Double
    public var isFrontSide: Bool
    public var texts: [TextResult]?
    public var fullImage: UIImage?
    public var croppedImage: UIImage?
    public var faceImage: UIImage?
    public var isFrontCardFull: Bool?
    public var classificationResult: IDCardClassificationResult?
  }
  ```

  - `error`: If the scanning is successful, the `error` will be null. In case of unsuccessful scan, the `error.errorMessage` will contain the problem.
  - `confidence`: A value between 0.0 to 1.0 (higher values mean more likely to be an ID card).
  - `isFrontSide`: A boolean flag indicates whether the scan found the front side (`true`) or back side (`false`) of the card ID.
  - `texts`: A list of OCR results. An OCR result consists of `type` and `text`.
    - `type`: Type of information. Right now, PreScreen support 3 types
      - `ID`
      - `SERIAL_NUMBER`
      - `LASER_CODE`
    - `text`: OCR text based on the `type`.
  - `fullImage`: A bitmap image of the full frame used during scanning.
  - `croppedImage`: A bitmap image of the card. This is available if `isFrontSide` is `true`.
  - `faceImage`: A bitmap image of the profile face on the card. This is available if `isFrontSide` is `true`.
  - `isFrontCardFull`: A boolean flag indicates whether the scan found the front side and the card is likely to be complete.
  - `classificationResult`: A result from ML classification.
    - `mlConfidence`: A float from 0.0 to 1.0 indicating the probability of an ID card.
    - `error`: A string, usually `nil` if no error.

### Full Example

```swift

import UIKit
import AVFoundation
import BrainergyPrescreen

class ViewController: UIViewController {

    var backFacingCamera: AVCaptureDevice?
    var frontFacingCamera: AVCaptureDevice?
    var currentDevice: AVCaptureDevice!

    var cameraPreviewLayer: AVCaptureVideoPreviewLayer?

    let captureSession = AVCaptureSession()

    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view, typically from a nib.
        Prescreen.shareInstance.initialize(apiKey: "ajMbRHTFPtUo9RzpSAMd")
        configure()
    }

    override func viewWillDisappear(_ animated: Bool) {
        captureSession.stopRunning()
    }

    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        // Dispose of any resources that can be recreated.
    }

    // MARK: - Helper methods
    private func configure() {
        captureSession.sessionPreset = AVCaptureSession.Preset.hd1280x720

        let deviceDiscoverySession = AVCaptureDevice.DiscoverySession(deviceTypes: [.builtInTelephotoCamera,.builtInWideAngleCamera], mediaType: AVMediaType.video, position: .unspecified)

        for device in deviceDiscoverySession.devices {
            if device.position == .back {
                backFacingCamera = device
            } else if device.position == .front {
                frontFacingCamera = device
            }
        }

        currentDevice = backFacingCamera

        guard let captureDeviceInput = try? AVCaptureDeviceInput(device: currentDevice) else {
            return
        }

        let output = AVCaptureVideoDataOutput()
        let queue = DispatchQueue(label: "myqueue")
        output.alwaysDiscardsLateVideoFrames = true
        output.setSampleBufferDelegate(self, queue: queue)

        captureSession.addInput(captureDeviceInput)
        captureSession.addOutput(output)

        cameraPreviewLayer = AVCaptureVideoPreviewLayer(session: captureSession)
        view.layer.addSublayer(cameraPreviewLayer!)
        cameraPreviewLayer?.videoGravity = AVLayerVideoGravity.resizeAspectFill
        cameraPreviewLayer?.frame = view.layer.frame


        captureSession.startRunning()

    }
    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        updatePreview()
      }

      func updatePreview() {
        let orientation: AVCaptureVideoOrientation
        switch UIDevice.current.orientation {
          case .portrait:
            orientation = .portrait
          case .landscapeRight:
            orientation = .landscapeLeft
          case .landscapeLeft:
            orientation = .landscapeRight
          case .portraitUpsideDown:
            orientation = .portraitUpsideDown
          default:
            orientation = .portrait
        }
        if cameraPreviewLayer?.connection?.isVideoOrientationSupported == true {
            cameraPreviewLayer?.connection?.videoOrientation = orientation
        }
          cameraPreviewLayer?.frame = view.bounds
      }
}



extension ViewController: AVCaptureVideoDataOutputSampleBufferDelegate {
    func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
        let result = Prescreen.shareInstance.scanIDCardSync(sampleBuffer: sampleBuffer, cameraPosition: self.currentDevice.position)
        if result.error != nil {
            print(result.error!)
        }
        else if (result.confidence >= 0.5) {
            print("Confidence: \(result.confidence)")
            print("Front side: \(result.isFrontSide)")
            print("Front side full: \(String(describing: result.isFrontCardFull))")
            if (result.texts != nil) {
                print(result.texts!)
            }
            if (result.isFrontCardFull == true) {
                // available if isFrontCardFull is true
                // print(result.croppedImage)
                if (result.classificationResult?.error == nil) {
                    print(result.classificationResult?.mlConfidence as Any)
                }

            }
        }
    }
}

```

## Requirements

This pod requires the following dependencies:

- GoogleMLKit/TextRecognition
- GoogleMLKit/ObjectDetectionCustom
- GoogleMLKit/ImageLabelingCustom

## Installation

BrainergyPrescreen is available through [CocoaPods](https://cocoapods.org). To install
it, simply add the following line to your Podfile:

```ruby
pod 'BrainergyPrescreen'
```

## Author

Brainergy, Info@brainergy.digital

## License

BrainergyPrescreen is available under the MIT license. See the LICENSE file for more info.
