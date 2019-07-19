# RailsChunkedFileUploads

This is a write up of using a Rails plug in to upload files through JQuery that are too large for a single file upload and must be split into chunks and reassembled on the server. It was originally posted on the UCSF CKM blog, but that is going away, so they're reposted here.

CHUNKED UPLOADS WITH JQUERY FILE UPLOAD AND RUBY ON RAILS
OCTOBER 21, 2014 GEOFF BOUSHEY
I was inspired by this thread on the Google Hydra-Tech forum to put together a short tutorial on chunked uploads with jQuery file upload and Ruby on Rails.

There’s a nice example on GitHub for using this plugin with Rails, so we don’t need to do this from scratch. To see how this plugin works without chunked file uploads, clone the example

https://github.com/tors/jquery-fileupload-rails-paperclip-example

bundle, rake:db, rails server, and go to port 3000. You should see a nice UI for file uploads.

Try uploading a small file (under 500k). It works well, has a nice looking upload bar that flashes for just a few moments, and allows you to delete or cancel an upload. Now try uploading something a little bigger, maybe around 500MB. The progress bar will really come in handy here, but once it is complete, your server will most likely hang for a few minutes (at least). A file a few GB in size, while permitted, will take even longer and may slow down your web browser and localhost server to the point where it is unusable, and you need to go off to terminate a few processes manually through the command prompt.

At some point, you’ll probably start wanting to use the chunked file uploads feature, which splits the file into smaller pieces and submits them, one after the other, across the wire to your server.

To activate chunked file uploads, open the jquery-fileupload-rails-paperclip-example app and navigate to the views/uploads/index.html.erb file. At the bottom of this file (line 118 on my system), you’ll see a call to:


$('#fileupload').fileupload();
To use chunked file uploads, replace the above bit of code with:


$('#fileupload').fileupload({
maxChunkSize: 1000000 
maxFileSize: 1000000 * 10000
});
Note – these values were chosen to illustrate the app working, you may want different values in production. For more on chunked file uploads, check the documentation.

Give it another try (you may need to reload the javascript for your change to take effect. Upload that medium size (at least a few dozen MB to see the full effect) and watch your dev server spin. You’ll notice that the controller for processing a file is getting called repeatedly for each chunk.

Watch your server log as rails processes the upload, and you’ll see that the controller is called repeatedly as you upload the file. The file uploader is splitting the medium sized file into 1 mb sized chunks and submitting them sequentially to the server.

Great, except.. refresh the page.

The rails app is processing each chunk as if it were a separate upload. Keep in mind, this has nothing to do with the jquery file uploader plugin, it’s doing exactly what we asked it to do – split up a file into small pieces and submit each one to the server. We just need to change how we’re processing this on the server side.

Before we go on, go ahead and delete those files (the fastest way is to check the box next to the upper menu delete button).

To process each chunk as part of a single file, rather than as a separate independent file upload, take a look at the create method in the uploads_controller.rb controller.

The first line


@upload = Upload.new(params[:upload])
creates a new upload object and processes it as a single file upload. You’ll need modify this to process it as a chunk rather than as a complete file.

There are plenty of ways to do this. To keep it all contained in an example, I’ll just go ahead and hack the controller. Remove the line above and replace it with the following code (entire method posted to avoid confusion about what to paste where)…



def create
    
     @temp_upload = Upload.new(params[:upload])

     @upload = Upload.where(:upload_file_name => @temp_upload.upload_file_name).first

     if @upload.nil?
       @upload = Upload.create(params[:upload])
     else
       if @upload.upload_file_size.nil?
         @upload.upload_file_size = @temp_upload.upload_file_size
       else
        @upload.upload_file_size += @temp_upload.upload_file_size
      end
     end

    p = params[:upload]
    name = p[:upload].original_filename
    directory = "uploads" 

    path = File.join(directory, name.gsub(" ","_"))
    File.open(path, "ab") { |f| f.write(p[:upload].read) }

    respond_to do |format|
      if @upload.save
        format.html {
          render :json => [@upload.to_jq_upload].to_json,
          :content_type => 'text/html',
          :layout => false
        }
        format.json { render json: {files: [@upload.to_jq_upload]}, status: :created, location: @upload }
      else
        format.html { render action: "new" }
        format.json { render json: @upload.errors, status: :unprocessable_entity }
      end
    end
  end
Note – to keep things simple, I hard coded a new upload directory into the method, so you’ll need a top level “uploads” directory in your rails app for this to work.

In this method, we are now reading the upload parameters into a temporary upload object. We then look for an upload object with that file name (yes, different uploads could have the same file name, so you’d need a different approach for production). If that object doesn’t exist, create it, and treat the file chunk as if it were a new upload. If it does exist, retrieve it and append the newly uploaded chunk to the existing file. You do this over and over until the last chunk is processed (there’s also a running counter for file size).

Give it a try and refresh the screen. This time, you should see only one file upload, and you should be able to retrieve that file from your uploads directory. You may also notice that the long lag time between the status bar completion and the actual file upload completion is much shorter now, as the file is getting written to disk in small increments (you don’t have to convert the entire temp file upload on the server at once after upload, just the last chunk).

One note – some of the functionality of the upload plugin (such as delete) will no longer work with the new directory location.

So, should I do chunked file uploads?

The general approach above, with some modifications, does make it possible to process bigger files and stick with a pure ruby solution, and it could take care of the problem of medium file uploads that need to be chunked but perhaps don’t require an industrial strength solution.

However, if you want to allow really big file uploads, you might want to consider a solution that allows a user to upload a file to box, dropbox, google drive, etc, and then transfer it to your server from there as a background job. In fact, there’s a very nice gem from hydra labs called “browse everything”  that provides this functionality and integrates nicely with sufia. If you’re already using the “browse everything” type approach, you might just go ahead and limit direct non-chunked uploads to small sized files that won’t tax the system rather than managing the complexity overhead of a solution that sits in between.

In addition to the obvious problems I dismissed with a bit of hand waving and vague excuses about this being for to a sample exercise, there are other issues to consider with large file uploads. What do you do about partial uploads? Partially uploaded files where the user closed the connection or lost the network connection? Checksums (we can check the sum on each chunk, but were the chunks assembled properly on the server)?

Chunked file uploads are a very useful way to get that middle ground, and I suspect many of your users would really appreciate being able to upload medium size files without having to create an external account. I just want to emphasize that will the above approach can work, there’s a lot to consider here.
