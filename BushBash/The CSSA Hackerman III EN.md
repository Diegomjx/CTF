![['https://github.com/Diegomjx/CTF/tree/master/BushBash/assets/Pasted image 20260803102215.png']]

Metadata

| Field            | Value                    |
| ---------------- | ------------------------ |
| ModifyDate       | 2000:01:01 00:00:00+0000 |
| YCbCrPositioning | Centered                 |
| ExifVersion      | 0210                     |
| DateTimeOriginal | 2000:01:01 00:00:00+0000 |
| CreateDate       | 2000:01:01 00:00:00+0000 |

---

To narrow down the search area, the first step is to determine the direction the camera was facing. Although the file metadata does not provide reliable time information, visual clues from the surroundings allow us to deduce the orientation: 

- Time of Day: The golden lighting and low solar altitude indicate that the photograph was taken during the late afternoon, close to dusk.
- Season: The presence of deciduous trees completely stripped of leaves places the scene in the middle of Canberra's winter (between June and August).
- Solar Analysis: During the southern winter, the Sun remains low in the sky and sets toward the Northwest.

To corroborate this, a solar trajectory tool ([alpenglowapp](https://alpenglowapp.com)) was used. By cross-referencing the lighting of the scene and the projection of the shadows with the sunset position, it was determined that the camera is oriented toward the southwest.


![[Pasted image 20260803110023.png]]

By cross-referencing the obtained orientation with Canberra's urban layout, it was identified that the only section of the city featuring aligned streets with views of buildings toward the **Southwest** is the following:

![[Pasted image 20260803113412.png]]

With the search radius narrowed down, the next step consisted of analyzing the specific urban infrastructure elements present in the image to achieve an exact geolocation:

![[Pasted image 20260803112509.png]]

A key detail in the image is the overhead power lines crossing over the street. Since this type of aerial cabling is uncommon in the modern areas of Canberra, its presence drastically narrows down the search range to a few specific points. Because these cables are difficult to identify from the standard satellite view, a specific traffic sign and the exact position of the tree beneath them were used as secondary references. After systematically inspecting the approximately 18 points with overhead wiring in the designated area, the exact location was found:

![[Pasted image 20260803115949.png]]
 [-35.278353,149.1443443]

The located site matches nearly all the visual elements of the original image, with a single exception: the presence of the current traffic lights. My hypothesis is that the analyzed photograph dates back to approximately 2022, a period prior to the installation of this traffic control infrastructure by the municipality. Except for that change, every other component confirms the geolocation unequivocally: the design of the building in the background, the structure of the adjacent house, the layout of the overhead cables, the tire tracks on the pavement, and the alignment of the vegetation (including the second tree, which clearly has not changed in size).


