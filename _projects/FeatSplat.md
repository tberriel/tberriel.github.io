---
layout: distill
title: Feature Splatting for Better Novel View Synthesis with Low Overlap
images:
  slider: true
  compare: true
description: 
tags: FeatSplat
giscus_comments: false
date: 2024-05-24
featured: true
authors:
  - name: Tomás Berriel Martins
    url: "https://tberriel.github.io/"
    affiliations:
      name: I3A, University of Zaragoza
  - name: Javier Civera
    url: "https://scholar.google.com/citations?user=j_sMzokAAAAJ&hl=en"
    affiliations:
      name: I3A, University of Zaragoza
bibliography: FeatSplat.bib

# Optionally, you can add a table of contents to your post.
# NOTES:
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - we may want to automate TOC generation in the future using
#     jekyll-toc plugin (https://github.com/toshimaru/jekyll-toc).

_styles: >
  .fake-img {
    background: black;
    color: white;
    border: 1px solid rgba(0, 0, 0, 0.1);
    box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);
    margin-bottom: 12px;
  }
  .fake-img p {
    font-family: monospace;
    color: white;
    text-align: left;
    margin: 12px 0;
    text-align: center;
    font-size: 16px;
  }
  .btn {
    background-color: black;
    color: white;
    text-decoration: none;
    padding: 10px 20px;
    border-radius: 5px;
    display: right;
    align-items: left;
    justify-content: center;
    transition: background-color 0.3s, color 0.3s;
  }

---

<a href="https://arxiv.org/abs/2405.15518" class="btn l-gutter" target="_blank" rel="noopener noreferrer">Paper <i class="ai ai-arxiv"></i></a>
<a href="https://github.com/tberriel/FeatSplat" class="btn l-gutter" target="_blank" rel="noopener noreferrer">Code <i class="fab fa-github"></i></a>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include video.liquid path="assets/video/both.mp4" class="img-fluid rounded z-depth-1" controls=true autoplay=true %}
    </div>
</div>
<div class="caption">
    Novel view synthesis of a trajectory on the scene 03f7a0e617 from ScanNet++ dataset<d-cite key="yeshwanthliu2023scannetpp"></d-cite>. Left 3D Gaussian Splatting<d-cite key="kerbl20233d"></d-cite>, right FeatSplat32.
</div>


3D Gaussian Splatting <d-cite key="kerbl20233d"></d-cite> has emerged as a very promising scene representation, achieving state-of-the-art quality in novel view synthesis significantly faster than competing alternatives. 
However, its use of spherical harmonics to represent scene colors limits the expressivity of 3D Gaussians and, as a consequence, the capability of the representation to generalize as we move away from the training views. 

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/publication_preview/featsplat.jpg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

In this paper, we propose to encode the color information of 3D Gaussians into per-Gaussian feature vectors, which we denote as Feature Splatting (FeatSplat). To synthesize a novel view, Gaussians are first "splatted" into the image plane, then the corresponding feature vectors are alpha-blended, and finally the blended vector is decoded by a small MLP to render the RGB pixel values. To further inform the model, we concatenate a camera embedding to the blended feature vector, to condition the decoding also on the viewpoint information.
Our experiments show that these novel model for encoding the radiance considerably improves novel view synthesis for low overlap views that are distant from the training views. Finally, we also show the capacity and convenience of our feature vector representation, demonstrating its capability not only to generate RGB values for novel views, but also to modify scene light after optimization, and to learn per-pixel semantic labels.


<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/gifs/path_2_pe.gif" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/gifs/path_2_x.gif" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/gifs/path_2_y.gif" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/gifs/path_2_z.gif" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Inference-time modification of scene light.
</div>



```scss
@article{martins2024feature,
      title={Feature Splatting for Better Novel View Synthesis with Low Overlap}, 
      author={T. Berriel Martins and Javier Civera},
      journal={British Machine Vision Conference (BMVC)},
      year={2024},
      eprint={2405.15518},
      archivePrefix={arXiv},
      primaryClass={cs.CV}
}
```
