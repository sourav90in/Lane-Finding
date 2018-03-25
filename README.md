# **Finding Lane Lines on the Road** 

## Overview

When we drive, we use our eyes to decide where to go.  The lines on the road that show us where the lanes are act as our constant reference for where to steer the vehicle.  Naturally, one of the first things we would like to do in developing a self-driving car is to automatically detect lane lines using an algorithm.

In this project you will detect lane lines in images using Python and OpenCV.  OpenCV means "Open-Source Computer Vision", which is a package that has many useful tools for analyzing images.


**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[image1]: ./examples/grayscale.jpg "Grayscale"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

My Pipeline consists of 6 steps:

1) The first step is the colour selection implemented via the color_select helper function.
Since the lanes were of either yellow or white colour, colour selection masks had to be able to detect yellow or white colours in the input images. From Udacity's lecture in Introduction to Comp Vision on Colour spaces, I got the idea to try converting the image from RGB to HSV format. This was because in all of the input images, the intensities of the yellow or white lanes weren't of consistent values, i.e. they faded in Values(Intensity) along the length of the image. Since this variation in a specific colour's intensity was easier to detect via the HSV model where changing the Saturation and Values for a specific Hue helps in obtaining different shades of the same colour, much easier as compared to RGB model where its easier to represent pure yellow or pure white conveniently.
  The first step was converting the image to HSV from RGB format. Next, the BGR values of Yellow(0,255,255) and White(255,255,255) were converted into their HSV equivalents. Once their HSV values were accessible, the next step was defining the lower and upper bounds for the colour mask for both yellow and white. I experimented with different varaitions of Hue, Saturation as well as Values for the lower and upper bounds for the Yellow and White Masks till satisfactory results were obtained for all the test-images, with the sky completely masked out and portions of the bushes surrounding the roads or the lane markings as well as the cars was the only content left over in the output of the masked image. Although ideally, I tried to filter out the bushes as well in the output, but couldn't manage to arrive at satisfactory results without losing out on important data related to yellow lane marking, so I didn't pursue that further and fixated on the lower and upper bounds for both the masks, as I believed that an accurate Region of Interest Selection would be able to concentrate on the area of focus i.e. the lanes and remove the rest of the content. The output masked color-selected image was converted back to RGB. 
  
2) The second step of the pipeline was the super-imposition of the color-selected image on top of the original image.
This step was inspired by the fact that the Canny Edge Detection algorithm would respond best to bigger changes in Intensity in the image as compared to smaller changes. Since converting the image to Grayscale for ease of convenience causes some dimming down of the image, it seemed like a good idea to take a weighted average of the color-selected image in RGB with the original RGB image to highlight areas of yellow or white more brightly.

In my initial implementation, I had converted the color-selected image to grayscale as well as the original image to grayscale and then taken a weighted average, but it seemed intuitively a good idea to take a super-position of the color selected image on top of the original image, before converting them to grayscale to avoid loosing out on any colour information.

3) The third step was conversion of the weighted image obtained in Step 2) into grayscale, followed by Gaussian blurring with a 7x7 kernel for intput to the Canny Edge Detection algorithm. In the initial implementation, I had not performed a gaussian blurring step, but doing the gaussian blurring steps helped in the canny edge detection step to miss out on irrelevant details like a portion of a car ahead in the lane or a portion of a bush along a curve while working on the test-videos. 

4) As part of the Canny Edge Detection algorithm, I had taken the initial lower and upper thresholds as 50 and 150 which were still giving good results on individual images. I later decreased the upper threshold to 130 to avoid missing out on sufficient details of non-solid lines of the frames of the video.

5) The next step was defining a Region of Interest to select on the output of the canny edge detection algorithm. Since the lanes were in the lower half of the image, a polygon of 4 sides seemed good enough to focus on the lane lines of interest. I kept altering the left top corner as well as the right top corner of the approximately looking trapezium region of interest till the inidividual frames in the video output for both the videos looke good enough to focus on mostly the lanes and not the cars around or the bushes. This was critical as any extra content left over from this step onwards to the Hough Lines algorithm would hinder its performance as the pollyffited left and right lanes in such a case would be off by a big margin. 
However even after altering the end-points of the region of interest consistently on single images, the performance was not too great, when the road curves in one of the videos. Eventually I decided to proceed with the output of this image and revert back to fine-tuning the end points of the region of interest as necessary depending on the performance of the draw_lines algorithm.

6) The next step was usage of Hough Lines to find line segments in the Region of interest. In the initial parameter setting for the Hough lines function, I had set the min line length as 20 pixels which I had to tone down to 15 to avoid missing out on dashed lane lines completely in some of the frames of the video, eventually. Once the line segments(XY pairs) were available from the HoughLinesP function, 

I came up with an initial draft of the draw_lines algo as below:

i) The draw_lines function had to find a way to distingiush between left and right lane line segments. The left lane segments would have a negative slope and the right lane segments would have a positive slope.

ii) The draw_lines function once able to differentiate between a class of negatively sloped line segments and positively sloped line segments, should be able to extend XY pairs i.e. X1,Y1 connected to X2,Y2, if having a similar slope as a line segment whose end-points are X3,Y3 and X4,Y4 should end up as a line segment that connects X1,Y1 to X4,Y4 with a slope that's an average of the slopes of both such line segments.

iii) Once all such positive and negative line-segments are connected, I would need to determine the length of the line representing the left lane as well as right lane i.e. the end points of the left and the right lane lines.

While implementing the first draft, it seemed counter-intuitive as to keep extending line segments into lines piece by piece since I had not exactly obtained a very good output from the above pipeline as some road-side corner bushes as well as car sections ahead in the lane were still not completely filtered out and joining XY pairs piece by piece would result in the actual lane wandering off in a completely wrong direction.

The next iteration of the idea was inspired by Linear Regression where we have a set of points and try out best to fit a line that passes through most of the points, so the implementation for draw_lines was refined as follows:

a) Classify line segments into two lists where one list would contain XY pairs representing a negative slope and the other list would contain XY pairs representing a positive slope.

After doing step a), I setup breakpoints in the code and had a look at typical values of slopes of line segments returned by the Hough Lines where most of the slopes were typically around -0.7 or around +0.5, which gave me an idea to leave out some XY pairs from the positive or negative lists, whose slope would be too much a departure from the mean slopes of each list.

b) Hence I calculated the mean slope of both the positive and the negative XY pairs list and removed XY pairs from both the positive and negative lists, which varied 0.15 from the mean slope in absolute values. This helped in removing out, some of the line segments that were still left over due to badly chosen parameters in the Hough Lines or the Canny Edge or the Region of Interest step.

c) The next step would be to find a poly fit of remaining XY pairs for both negative and positive lists that would give the Line parameters(slope and intercept) of the lines that would be able to represent the left of the right lanes.

d) The next step would be determine the start and end points of each of those lines in the image. Since the lanes were in the lower half of the image, I set the starting Y coordinate of the lines as the end of the image in Y axis and the ending Y-coordinates of the image, close to the middle of the image.

e) Once the Start and end Y-coordinates of the lines was fixated, the X-coordinates could be simply found by using the roots function in the Polynomial library and once the X-coordinates and Y-coordinates were available for each of the lines, the lines could simply be drawn on the image.

While the above iteration of the idea worked good for the sample images, running the same on the video posed two issues:
i) The algo sometimes crashed which upon debugging indicated that some of the slopes of the line segements that were returned by the Hough function were of Inf value.
ii) The lines representing the left and right lanes were too shaky and sometimes not adhering to the lane boundaries and shooting off in random directions.

In order to address the above issues, I added debug points as well as stored the outputs of the Region of Interest step and the draw_lines output to a folder and observed that:
a) Some of the line-segments from the HoughLinesP function were resulting from the surrounding bushes or cars ahead in the lane or because the road was curving and the region of interest was so big that the output of Step 5), looked to imply to the HoughLinesP function that both the lanes were intersecting when the road looked to curve.
b) Most of the line-segmegents had slopes around -0.7 and around 0.5 and very few had slopes greater than 1 or smaller than 1.

In order to mitiage the completely vertical line segments(infinite slope), I narrowed down the region of interest and thresholded out some of the line-segments that had values greater than +1 or lesser than -1.
In order to further reduce the set of line-segments that would cause a deviation in the mean-slope calculation slope of the positive and negative lists, I further dropped some line-segments that were within the -0.25 to +0.25 range.

In order to stabilize the lines representing the left and right lanes for the video frames, I realized I could make some usage of temporal data across video-frames where the X coordinate start points of the left and right lane lines could depend on the X start points of the lines representing the left and the right lanes. This helped in trememdously stablizing the drawn lines across video frames especially at the points of the video where the road would curve.

7) The last and final step was to obtain a weighted combination of the Hough Lines function's image output and the actual image which would give the left and right lane markers super-imposed on top of the original image.

The output test-results of the Test-Images are uploaded to the directory:
https://github.com/sourav90in/Lane-Finding/tree/master/test_images_output

The output test-results of the Videos are uplaoded to the directory:
https://github.com/sourav90in/Lane-Finding/tree/master/test_videos_output

### 2. Identify potential shortcomings with your current pipeline


One potential shortcoming would be what would happen when the road curves continuosly or when there are regions of the road that
have tremendous differences in illumination. Huge differences in sunlight based illumination or a different road-colour would certainly throw off the colour-selection step as it would be highlight such regions more prominently to the Canny Edge Detection step which can possibly detect a big number of false edges.

Also this problem was highlighted when I tried out the Challenge Video where the line representing the Right lane underwent a  rotation of 90 degrees before recovering when the patch of the road was encountered that had a yellowish tinge to it. The outoput of the Challenge Video and the problem that surfaced in it, can be viewed in the Video uploaded to the test_videos_output folder.

Also when the road gets too curvy, the idea of utilizing temporal data across frames of a video falls apart since the X-coordinate start and end points of the lines representing the left and right lanes would be uncorrelated across video frames.

### 3. Suggest possible improvements to your pipeline

I could have done a better task at selecting better thresholds for the Color Selection masks which would be able to filter out changes in road-colour. Also a better choice of Hough transform parameters with respect to the size of the bins in Hough Space could possibly result in fewer irrelevant line segments from being detected.
Another possible improvement would be have a feedback loop that alters the size of the Region of Interest when the slope of the lines representing the left and right lanes changes rapidly across a space of 5 frames.
