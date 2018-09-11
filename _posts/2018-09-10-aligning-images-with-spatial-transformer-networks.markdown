---
layout: post
use_math: true
---
# Motivation
To demonstrate the capabilities of the [Inverse Compositional Spatial Transformer Network (ICSTN)](https://chenhsuanlin.bitbucket.io/inverse-compositional-STN/paper.pdf), I trained a network to predict the warp parameters between a warped and unwarped image. The network takes two images, an unmodified image and an image modified by an affine transform and returns the warp parameters believed to cause the transformation.

# Dataset creation
I created a dataset by taking 64x64 center crops from the [SceneNet dataset](https://robotvault.bitbucket.io/scenenet-rgbd.html)
 and generating a random warp to create 170k image-warp pairs. The noise parameter was set to 0.1 for training.

Example images:

| Unwarped | ![unwarped_example_0](/assets/scenenet/unwarped_0.png) | ![unwarped_example_2](/assets/scenenet/unwarped_2.png) | ![unwarped_example_6](/assets/scenenet/unwarped_6.png) | ![unwarped_example_5443](/assets/scenenet/unwarped_5443.png)
| Warped | ![warped_example_0](/assets/scenenet/warped_0.png) | ![warped_example_2](/assets/scenenet/warped_2.png) | ![warped_example_6](/assets/scenenet/warped_6.png) | ![warped_example_5443](/assets/scenenet/warped_5443.png)

# Network Architecture and Training

The network architecture is loosely based on VGG Net. The two images go through independent convolutional layers with shared parameters, join and go through shared convolutional layers, and finally through fully connected layers to predict the warp parameters.

I train the network using the Adam optimizer with batch size 32, and an initial learning rate of 1e-3, decaying with a gamma of 0.75 every 20 epochs. Total training time is 100 epochs.

Network architecture:

```python
class AlignNetwork(nn.Module):
    """ Applies an IC-STN transformation to the incoming image
        
        Args:
            in_channels: the number of channels in the image
            num_compositions: the number of recurrent compositions to perform
            warp_type: the type of transformation to learn. (translation, similarity, affine, homography)
    """

    TRANSLATION = 'translation'
    SIMILARITY = 'similarity'
    AFFINE = 'affine'
    HOMOGRAPHY = 'homography'
    
    def __init__(self, in_channels, num_compositions, warp_type, device):
        super(AlignNetwork, self).__init__()

        self.in_channels = in_channels
        self.num_compositions = num_compositions
        self.warp_type = warp_type
        self.device = device

        if self.warp_type == AlignNetwork.TRANSLATION:
            self.warp_dim = 2
        elif self.warp_type == AlignNetwork.SIMILARITY:
            self.warp_dim = 4
        elif self.warp_type == AlignNetwork.AFFINE:
            self.warp_dim = 6
        elif self.warp_type == AlignNetwork.HOMOGRAPHY:
            self.warp_dim = 8
        else:
            raise ValueError('warp_type must be one of '
                '(translation, similarity, affine, homography)')

        # Together, the convolutional and fully connected layers take
        # in an image and predict a delta_p which gets inversely composited
        # with the current warp parameters, p

        self.target_conv_layers = nn.Sequential(
            nn.Conv2d(self.in_channels, 64, kernel_size=[3,3], padding=get_padding_size(3)),
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.Conv2d(64, 128, kernel_size=[3,3], padding=get_padding_size(3)),
            nn.BatchNorm2d(128),
            nn.ReLU(),
            nn.MaxPool2d([2,2], stride=2),
            nn.Conv2d(128, 256, kernel_size=[3,3], padding=get_padding_size(3)),
            nn.BatchNorm2d(256),
            nn.ReLU(),
            nn.Conv2d(256, 256, kernel_size=[3,3], padding=get_padding_size(3)),
            nn.BatchNorm2d(256),
            nn.ReLU(),
            nn.MaxPool2d([2,2], stride=2),
        )

        self.combined_conv_layers = nn.Sequential(
            nn.Conv2d(512, 512, kernel_size=[5, 5], padding=get_padding_size(5)),
            nn.BatchNorm2d(512),
            nn.ReLU(),
            nn.Conv2d(512, 512, kernel_size=[5, 5], padding=get_padding_size(5)),
            nn.BatchNorm2d(512),
            nn.ReLU(),
            nn.MaxPool2d([2,2], stride=2)
        )

        self.param_layers = nn.Sequential(
            nn.Linear(512 * 8**2, 2048),
            nn.BatchNorm1d(2048),
            nn.ReLU(),
            nn.Linear(2048, 1024),
            nn.BatchNorm1d(1024),
            nn.ReLU(),
            nn.Linear(1024, self.warp_dim)
        )


    def forward(self, warped_image, unwarped_image, return_p=False):
        image_warped_all = [] # keep track of all the warped images across time
        warps = []
        batch_size = unwarped_image.size(0)
        p = torch.zeros(batch_size, self.warp_dim).to(self.device)

        target_im_feats = self.target_conv_layers(warped_image)

        for i in range(self.num_compositions):
            p_matrix = vector_to_matrix(p, self.warp_type)
            image_warped = transform_image(unwarped_image, p_matrix)
            image_warped_all.append(image_warped)
            warps.append(p_matrix)

            feat = self.target_conv_layers(image_warped)
            im_conv_feats = torch.cat([feat, target_im_feats], 1)
            combined_feats = self.combined_conv_layers(im_conv_feats)
            delta_p = self.param_layers(combined_feats.view(batch_size, -1))
            p = compose(p, delta_p, self.warp_type)

        p_matrix = vector_to_matrix(p, self.warp_type)
        image_warped = transform_image(unwarped_image, p_matrix)
        image_warped_all.append(image_warped)

        if return_p:
            return image_warped_all, matrix_to_vector(p_matrix, self.warp_type)
        else:
            return image_warped_all
```

![train_loss](/assets/scenenet/train_loss.png)

![epoch_loss](/assets/scenenet/epoch_loss.png)

# Evaluation