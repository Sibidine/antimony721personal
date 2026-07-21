
### 3D gaussian splatting[^1]

This is a method of novel view synthesis, where, from photos or videos captured of a scene, novel views from unseen angles can be constructed of that scene. 

The input to this method is a set of images of a static scene. From these images, corresponding cameras are calibrated using SfM[^2]. This calibration of cameras also produces a sparse-point cloud as a side effect.

#### Creation of the Sparse Point Cloud

Distinct 2D points of interest are identified in every individual image. The system compares these features across the entire image collection to establish matching correspondences, which identify where different images observe the exact same exact scene points. 

After establishing the above, we come to reconstruction. Two initial, overlapping images are selected, and the relative distance and orientation between their camera views are estimated. By analysing the matched 2D features between the two cameras, it calculates the 3D position of those specific points (this being an inital fragment of the sparse 3D point cloud).

For the calibration of the next camera, a mathematical process called *Perspective-n-Point*  (PnP) is followed: it looks at the 2D features in the new image and identifies which ones correspond to the already existing 3D points in the initial point cloud (this is known as _2D-3D correspondence_). By aligning the camera such that these 3D points correctly project onto their observed 2D pixels, the camera's precise pose is mathematically locked into place.

Once this new camera's position is successfully calibrated, the system uses it to discover new structure. The newly registered camera will comtain 2D feature matches corresponding to other registered images that habe not yet been mapped into 3D space. Rays are projected from the new and old cameras through these matched 2D pixels, and the exact point where they intersect in 3D space is found. 

This geometric instersection calculation, called triangulation, creates brand new 3D coordinates and adds them to the existing structure, progressively expanding the size of the sparse point cloud. 

#### Defining the 3D Gaussians

Coming to the 3D Gaussians, they are defined by a full 3D covariance matrix $\Sigma$ defined in world space, centred at point (mean) $\mu$:
$$G(x) = e^{-\frac{1}{2}(x)^T\Sigma^{-1}(x)}$$

, which is then multiplied by $\alpha$ (opacity) in the blending process.
To project these 3D gaussians into 2D, given a viewing transformation W, the covariance matrix $\Sigma'$ in camera coordinates is given as:
$$\Sigma' = JW\Sigma W^TJ^T$$
where J is the Jacobian of the affine approximation of the projective transformation. 
The covariance matrix, for purposes of keeping it positive semi-definite through gradient descnet, is paramtereized by the scaling matrix S and rotation matrix R in the following manner: 
$$\Sigma = RSS^TR^T$$
, with a 3D vector s for scaling and a quaternion q for rotation being stored separately, such that they may be converted to their respective matrices and combined trivially.

#### Optimization

Stochastic Gradient Descent is used for optimization, since it allows us to create, destroy to move geometry. A sigmoid activation is used to constrain opacity, and an exponential activation function for the scale of the covariance. 

The loss function is $\mathcal{L}_1$ combined with a /d-SSIM term:
$$\mathcal{L} = (1-\lambda)\mathcal{L}+\lambda\mathcal{L}_{D-SSIM}$$
 with $\lambda$ = 0.2.

#### Adaptive Control

This focuses on areas with large view-space positional gradients- which may be missing geometric features (under-reconstruction) and regions where Gaussians cover large areas (over-reconstruction).In the case of small Gaussians in under-resconstructed areas, the Gaussians are cloned, and for large Gaussians with with high variance, spltting is done: they are replaced by two new Gaussians, and their scale is divided by a factor of $\phi = 0.6$. 

To avoid getting too many Gaussians, the opacity $\alpha$ is set close to zero every N = 3000 iterations, and Gaussians with $\alpha < \epsilon_{\alpha}$ are culled.

#### Fast Differentiable Rasterizer

This rasterization is treated as a continuous mathematical function rather than a set of hard pixel assignments. This is achieved through differentiable alpha blending- the rendered looks at the splats in order of front to back, accumulating colours based on opacities.

3DGS uses a rasterizer having the following steps:
1. the system identified and discards any Gaussians that are completely outside the camera's view.
2. It divides the screen into a grid of small tiles, 16 $\times$ 16 in this case. 
3. Which Gaussians land in which tiles is calculated. In case a gaussian overlaps over multiple tiles, it is virtually duplicated and assigned a key for each tile.
4. A radix sort is done to organise the Gaussians on each tile according to their depth, and thus pixel-wsie sorting is avoided
5. Front to back colour and opacity blending is done.

## iVR-GS

This extends 3D-GS to volume visualization, and improves reconstruction quality and composability.[^3]

#### Viewpoint Sampling 

To generate multi-view images required for training, designing a set of basic transfer functions to extract meaningful objects from the VolVis scene training is necessary. Icosphere samplng is used 12 multi-view images for each basic transfer function as initial sample images. The complexity of the basic scene is evaluated to determine the number of viewpoints for training. This complexity is calculated through a combination of colour and opacity entropies over initial sample images, and is given by 

$$E = -\Sigma_{i = 1}^{N_p}(p_i^c\ log\ p_i^c + p_i^\alpha\ log\ p_i^\alpha)$$

Where N<sub>p</sub> is the total number of pixels for all initial sample images, $p_i^c$ and  $p_i^\alpha$ represent the i-th pixel's colour and alpha value probabilities respectively. This score is normalized by dividing it by the max entropy score among all basic scenes. Varying number of additional views are sampled uniformly beyond this if necessary.

#### Base 3DGS Optimisation

The version of 3DGS used here has an extra normal attribute **n** incorporated for each Gaussian, and is optimized[^4] as:

$$\{D,N\} = \Sigma_{i \in N}T_i\alpha_i\{d_i, n_i\}$$
where D and N denote the rendered depth and normal maps of the base 3DGS represented scene, $d_i$
 and $n_i$ are the depth and normal of the i-th sampled Gaussian. After this, a pseudo normal $\widetilde{N}$ from D is calculated under the local planarity assumption, and the normal consistency loss is:
 $$\mathcal{L} = ||N - \widetilde{N} ||_2 $$ 
A collection of SH coefficients are also used to resore the colour information of each Gaussian. Optimizing **n** is to provide an initialzation that can accelerate the converge of editable Gaussian optimization. The above loss is used alongside the standard L-1 and SSIM losses for base 3DGS output colour and alpha channels.

After optimization, the i-th Gaussian is parameterized by $\{\mu_i, q_i, s_i, o_i, n_i, c_i\}$, where the view dependent colour c_i is represented by SH coefficients.

>[!Note]
>The final size of iVR-GS outputs is smaller than that of 3dGS even though the spherical harmonics of the latter seem to be present in the former. This is because iVR-GS uses Blinn-Phong reflection for colour, and the spherical harmonics (SH) coefficients are not present in the final  model when saved in  phong  mode.


#### Editable Gaussian Optimisation

The Blinn-Phong reflection model is used to compute the view-dependent colour of each Gaussian instead of using SH coefficients.  Additional shading attributes are assigned to each Gaussian- an offset colour $\Delta c \in \mathbb{R}^3$, ambient, diffuse and specular coefficients $k^a, k^d, k^s$ and a shininess term $\beta$. 
Additional regularizations to optimise the opacity term and shading attributes are also used- low opacity Gaussians are periodically removed and opacity of Gaussians is kept at a minimum using L1 regularization. Sparsity regularizatio  is also added to avoid $\Delta c$ causing significant palette colour shiftings, which is defined as the L1 loss of the rendered offset colour map. Bilateral smoothness regularization is employed for the Phong coefficients and shading attributes. 

After this step, the i-th Gaussian is parameterized by  $\{\mu_i, q_i, s_i, o_i, n_i\}$, along with shading attributes  $\{\Delta c_i, k^a_i, k^d_i, k^b_i, \beta_i\}$.

#### Compression Via Vector Quantization

k-means Clustering is used to generae a shared codebook for one attribute value across all Gaussians. Instead of storgin the exact value of an attribute, each primitive stores an index to the nearest value within the fixed size codebook. Such codebooks are generaed for all editable Gaussian attributes except $\mu$ and **n** since quantizing these lowers reconstruction quality.

#### Inverse Volume Exploration

Given a reference point and corresponding camera pose, the parameters of all editable Gaussian attributes ae frozen, and transformation parameters for colour, opacity and lighting for each basic iVR-GS model is learnt. 







---
[^1]: Kerbl, Bernhard, et al. "3d gaussian splatting for real-time radiance field rendering." _ACM Trans. Graph._ 42.4 (2023): 139-1.
[^2]: Schonberger, Johannes L., and Jan-Michael Frahm. "Structure-from-motion revisited." _Proceedings of the IEEE conference on computer vision and pattern recognition_. 2016.
[^3]:  Tang et al. - iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting
[^4]: J. Gao, C. Gu, Y. Lin, H. Zhu, X. Cao, L. Zhang, and Y. Yao. Relightable 3D Gaussian: Real-time point cloud relighting with BRDF decomposition and ray tracing. arXiv preprint arXiv:2311.16043, 2023. doi: 10.48550/ arXiv.2311.16043 2, 4, 7


 