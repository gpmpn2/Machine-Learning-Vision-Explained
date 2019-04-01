# Machine Learning Vision: Explained
 
#### ImagePickerController
```
let imagePicker = UIImagePickerController()
```
* Creating the image picker controller
* https://developer.apple.com/documentation/uikit/uiimagepickercontroller
```
imagePicker.delegate = self
textView.text = ""
activityIndicator.hidesWhenStopped = true
```
* Setting the delegate of the imagePicker to the parent viewcontroller
* Clearing the current textview, as nothing has been done yet
* Set the activity indicator to hide itself onces its task has been completed
```
@IBAction func cameraSelected(_ sender: Any) {
    takePhotoWithCamera()
}
```
* Creating an action to the camera option button, when clicked it will execute the method to bring up the camera
```
@IBAction func photoLibrarySelected(_ sender: Any) {
    pickPhotoFromLibrary()
}
```
* Creating an action to the photo library optino button, when clicked it will execute the method to bring up the photo library
```
func takePhotoWithCamera() {
    if (!UIImagePickerController.isSourceTypeAvailable(UIImagePickerController.SourceType.camera)) {
        let alertController = UIAlertController(title: "No Camera", message: "The device has no camera.", preferredStyle: .alert)
        let okAction = UIAlertAction(title: "OK", style: .default, handler: nil)
        alertController.addAction(okAction)
        present(alertController, animated: true, completion: nil)
    } else {
        imagePicker.allowsEditing = false
        imagePicker.sourceType = .camera
        present(imagePicker, animated: true, completion: nil)
    }
}
```
* If the device in use has a camera available, set the source type and disallow editing of the camera, and present the camera. If not, send an alert notifiying the user that there is no camera to use.
```
func pickPhotoFromLibrary() {
    imagePicker.allowsEditing = false
    imagePicker.sourceType = .photoLibrary
    present(imagePicker, animated: true, completion: nil)
}
```
* Sets the image picker to disallow editing, and make its source the photoLibrary. Then present the imagepicker. No need to check if there is a camera on the device here because all devices have a photo library.
```
func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
    if let pickedImage = info[UIImagePickerController.InfoKey.originalImage] as? UIImage{
        imageView.contentMode = .scaleAspectFit
        imageView.image = pickedImage
        textView.text = ""
            
        guard let ciImage = CIImage(image: pickedImage) else {
            displayString(string: "Unable to convert image to CIImage.");
            return
        }
            
        detectScene(image: ciImage)
    }
        
    dismiss(animated: true, completion: nil)
}
```
* This function is overrided as the delegate was set in the viewdidload. That is why there is no override keyword infront of it.
* First it checks if the image picked (or taken by camera) is an actual UIImage, if so lets process it.
* It sets the imageView to scaleAspectFit (AKA determines the way the image will display), it sets the imageView to the image taken and then clears the textfield of the description.
* Next the method attempts to create a CIImage out of the pickedImage. The CIImage is used to do Machine Learning (image processing) on the image. The reason it is guarded is because if it fails to create a CIImage the image processing cant be done. And therefore we must break out of the method.
* Next the method will call the detectScene function on the given CIImage
* Lastly we dismiss the imagepicker
```
func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
    dismiss(animated: true, completion: nil)
}
```
* If the user clicks the cancel button when selecting an image or taking a picture, dismiss the image picker (delegate function)
```
func displayString(string: String) {
    textView.text = textView.text + string + "\n";
}
```
* Sets the summary action textView's text content to the given string passed in
```
func detectScene(image: CIImage) {
    displayString(string: "detecting scene...")
        
    // Load the ML model through its generated class
    guard let model = try? VNCoreMLModel(for: VGG16().model) else {
        displayString(string: "Can't load ML model.")
        return
    }
        
    // Create a Vision request with completion handler
    let request = VNCoreMLRequest(model: model) { [weak self] request, error in
        guard let results = request.results as? [VNClassificationObservation],
            let _ = results.first else {
                self?.displayString(string: "Unexpected result type from VNCoreMLRequest")
                return
        }
            
        // Update UI on main queue
        DispatchQueue.main.async { [weak self] in
            self?.activityIndicator.stopAnimating()
            for result in results {
                self?.displayString(string: "\(Int(result.confidence * 100))% \(result.identifier)")
            }
        }
    }
        
    activityIndicator.startAnimating()
        
    // Run the Core ML GoogLeNetPlaces classifier on global dispatch queue
    let handler = VNImageRequestHandler(ciImage: image)
    DispatchQueue.global(qos: .userInteractive).async {
        do {
            try handler.perform([request])
        } catch {
            DispatchQueue.main.async { [weak self] in
                self?.displayString(string: error.localizedDescription)
                self?.activityIndicator.stopAnimating()
            }
        }
    }
}
```
#### Line by line analysis
* Sets the summary action textview to saying "detecting scene..."
* Attempt to load the machine learning model VGG16(), if it fails the method returns (guard else)
* Since we have the model now, the method makes a vision request. This function call has a completion handler of a request and an error. We are assigning this to a VNCoreMLRequest object called "request"
* Guard else statement to first see if we can get the "request" result as a VNClassificationObservation. If so we then do another check to see if we can get the first of these results. If this fails, we set the summary action text view to display a error.
* If we now have all of this information, we will dispatch the main queue. Once the main queue has free time to complete this task it will stop our activity indicator (notifiying the user the image processing is done), and fill the textView with the processed information
* We are now looking at the lines outside of the "request" object.
* We start animating the activity indicator
* We create a handler for the CIImage passed in that was taken from either a camera or photo library
* Next we create a Global Dispatch Queue that is asynchronous. This will get our data we need without disrupting anything else currently running on a queue. 
* Since we will be updating the UI once this process is done (AKA The DispatchQueue.main inside the "request" object, we are using a Quality of Services of .userInteractive)
* Also note this is wrapped in a try, catch because the handle.perform function can possibly throw an error. In the case it DOES throw an error we stop the activity indicator and display the error.
