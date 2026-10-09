## Minecraft District Classifying Model
**Repository:** https://github.com/mustafa-5500/Minecraft-Districting-Model
**Tools:** 
- **Minecraft-Git Wrapper:** https://github.com/mustafa-5500/Minecraft-Git-Plugin
- **Minecraft-Building Lore/Labelling:** https://github.com/mustafa-5500/Minecraft-Building-Lore-Linking-Plugin

## Table of Contents
- [[#Overview]]
- [[#Potential Architectures]]
- [[#Data]]
- [[#Project Proposals]]

## Overview
**Goal:** Create a Classification model which is able to take 3D block and entity data of any size, classify the types of building/structures in the data.

- **Primary Classification:**
	- What the building/structure is.
	- What architectural style the building/structure follows.
	- What the bounding 3D area is for that building or structure.

- **Secondary Classification:**
	- Then classifying groups of adjacent or nearby structures as districts based on shared semantic meaning.
		- The training data will also include descriptions (lore) for structures, so the model will need some level of natural language processing.
		- For predictions the model might have access to description (lore) if the player/user set it. But it should be able to predict some semantic meaning behind structures, without needing their descriptions.

## Potential Architectures:
- **Convolutional Neural Network:** Represent the buildings as 3D tensors, then use a 3D tensor as a kernel, a minimum size would be 3 by 3, with stride 1, to account for all neighbouring blocks. Since the bounding box of buildings does not follow a cuboid shape, rather it is a complex shape constructed from a set of cuboids. The model will have some cases where sections of the kernel are processing on empty data. Furthermore we should follow/be inspired by the R-CNN object detection architecture.
	- **Question:** Can a Convolutional Neural Network be resolution Agnostic? Meaning not being restricted to a certain resolution. #Experiment 
- **Convolution Next-Voxel Prediction Model:** Similar to Language Models, perhaps the model can predict what block comes next. In this case next isn't ordering, but rather what block exists in a relative voxel coordinate to the given blocks. Or maybe instead of the blocks it predicts what the class of the next voxel coordinate is given the blocks and nearby classified voxel coordinates? #Experiment  

**Input Dimensions:**
Let $\mathbf{x}$ be a tensor of the shape $w\times h\times d\times 870$ which represents any 3D space in a minecraft world, with each voxel being filled with one of 870 potential blocks, the $\vec0$ will represent air.
**Question:** Should Air be another value rather than $\vec{0}$? #Experiment 
-  What if there exists voxel buildings which are disconnected? Then it might be beneficial to distinguish between Air and null, since null will be used to pad the inputs, therefore null values can also be used as an indication that the prediction for an entire building is complete.

We would like to create a model which is not restricted to a pre-defined $w\times h\times d$. So the model has two options of dealing with generalized sized inputs.
- **Compression:** Recursively compress the input size until it reaches within the bound of a pre-defined size, then classify using the compressed size.
- **Cropping:** Split the input into the pre-defined size and then classify the voxels.

**Behaviour:**
The model is trying to semantically segment the 3D voxel space, therefore it should have precision as little as one voxel, so for minecraft, it will be one block. Furthermore it would want to split semantic segments per each individual instance, so multiple real instances should not be predicted as one instance. Problematically buildings can share voxels, so some voxels will correctly have to be predicted as being part of two instances. 
**Question:** How can a convolution network have classification precision of 1 voxel? #Experiment 
**Question:** How can a network be trained with voxels sharing classification? So non-independent classification? #Experiment 
**Question:** How can the network learn to properly divide nearby and neighbouring voxels of the same class as being part of different instances? #Experiment 

**Output Dimensions:**
The output will be a collection of multiple one-hot vectors. Let $\mathbf{s}$ represent the vector for architectural styles. Let $\mathbf{t}$ represent the vector for building type. Let $\mathbf{e}$ represent the vector for environment/biome. So the output will be a matrix $\mathbf{Y} = \begin{bmatrix}\mathbf{s}, \mathbf{t}, \mathbf{e}\end{bmatrix}$. Since the three vectors might not have the same dimensions, we will pad them so they can fit into one matrix.

**Question:** Why not make the 3 vectors into a 3D tensor? #Experiment 
## Data
- **Input:** 3D Block and entity data, represented as groups of 4 csv files for each plane on the vertical axis, each csv file representing one quadrant of each plane. Allowing the first entry of every csv file to be the origin of the plane. The block data is text describing what block exists in the position, and any modifiers on that block such as waterlogging or colouring.
	- **Note:** This data is discrete values, so having them represented as different numbers along the same dimension might not work. 
		- **Naive-Solution:** Represent each coordinate with a one-hot vector which is the size of all possible blocks, so ~870. But this makes our data 4D.
		- **Embedding Matrix:** To reduce the dimensionality of the one-hot vectors we can represent them as values within smaller vectors. This still leads to 4D input data. Could be used with a block gradient and shape map, so the different colours will be different dimensions, and each shape a block can take will be its own dimension.
		- **Single-Value Encoding:** Find a function to represent every block as a single value, perhaps have similar blocks placed near eachother?

- **Labels:** Text representing the building type, and the architectural style. Followed by a set of min & max corners to cuboid regions, Therefore representing a set of cuboid regions, which can represent any 3D block area. The 3D area is the bounding area of the label.
	- **Idea:** for the label it could be a multi-dimensional label, for example building-type x architecture-style x building-biome. So we can have a Persian home in a desert.
		- **Question:** Will increasing dimensionality of the output reduce the negative effects of information compression? #Experiment
			- Technically this multi-dimensional label is still a one hot vector, however along multiple dimensions. So there is still a lot of compression happening
	- **Secondary Label:** Description/lore for bounding areas, the descriptions will not follow from a pre-determined list as the labels do, instead the descriptions are natural language.

- **Data:** Currently we have the following cities with architectural style and approximate building counts, in the form of placeable in-game blocks, and any additional modifiers:
	- **Blocks:** ~870 placeable blocks.

	- **Buildings:**
		- **Note:** The counts are approximate, the data needs to be normalized & counted, as well as split into each individual building.
	- The Free City, Qadakh – Persian architecture, 200+ buildings
	- Ar’Kath, Qadakh – Terracotta architecture, 30+ buildings
	- Hyluse, Qadakh – Mediterranian architecture, 30+ buildings
	- Port Nici, Zaravento – South East Asian architecture, 40+ buildings
	- The Vale – Medieval European architecture, 30+ buildings
	- Mesa Town, Qadakh – Terracotta architecture, 10+ buildings
	- Port Mia, Zaravento – Spanish architecture, 10+ buildings
	- Port Attas, Zaravento – Spanish architecture, 10+ buildings
		- Total: 360+ buildings, 6 Styles

	- **Data Augmentations:** To increase our data set, we can apply augmentations which do not effect the semantic meaning of the buildings, such as changing orientation, and changing colour tints (for blocks with this attribute, such as water and foliage)

	- **Other Data:** The model will also need to differentiate buildings from wilderness, randomness, and ruins.
		- **Ruins:** Hestia, Qadakh – 20+, Desert Ruins
		- **Wilderness:**
			- **Biomes:** 95+ biomes (terralith data pack) X (potential) 3 elevations, flat, hilly, mountainous
			- **Trees:** Unknown amount of terralith trees, + custom player built trees.
		- **Randomness:** Can be generated given a list of blocks.

## Project Proposals
Since the current available data will only add upto to a couple thousand, after applying data augmentation, it is likely we wont have enough data to get interesting results. Furthermore processing the current data has a high labour cost, so it is not realistic to have it done in less than 2 months. Therefore it is more realistic to work on public data sets such as ImageNet, and attempt some of our experiments.
