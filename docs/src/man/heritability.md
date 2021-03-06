
# Heritability Analysis

As an application of the variance component model, this note demonstrates the workflow for heritability analysis in genetics, using a sample data set `cg10k` with **6,670** individuals and **630,860** SNPs. Person IDs and phenotype names are masked for privacy. `cg10k.bed`, `cg10k.bim`, and `cg10k.fam` is a set of Plink files in binary format. `cg10k_traits.txt` contains 13 phenotypes of the 6,670 individuals.


```julia
;ls cg10k*.*
```

    cg10k.bed
    cg10k.bim
    cg10k.fam
    cg10k_traits.txt


Machine information:


```julia
versioninfo()
```

    Julia Version 0.5.1
    Commit 6445c82 (2017-03-05 13:25 UTC)
    Platform Info:
      OS: macOS (x86_64-apple-darwin13.4.0)
      CPU: Intel(R) Core(TM) i7-4790K CPU @ 4.00GHz
      WORD_SIZE: 64
      BLAS: libopenblas (USE64BITINT DYNAMIC_ARCH NO_AFFINITY Haswell)
      LAPACK: libopenblas64_
      LIBM: libopenlibm
      LLVM: libLLVM-3.7.1 (ORCJIT, haswell)


## Read in binary SNP data

We will use the [`SnpArrays.jl`](https://github.com/OpenMendel/SnpArrays.jl) package to read in binary SNP data and compute the empirical kinship matrix. Issue 
```julia
Pkg.clone("https://github.com/OpenMendel/SnpArrays.jl.git")
```
within `Julia` to install the `SnpArrays` package.


```julia
using SnpArrays
```


```julia
# read in genotype data from Plink binary file (~50 secs on my laptop)
@time cg10k = SnpArray("cg10k")
```

     33.883711 seconds (282.07 k allocations: 1015.002 MB, 0.23% gc time)





    6670×630860 SnpArrays.SnpArray{2}:
     (false,true)   (false,true)   (true,true)   …  (true,true)    (true,true) 
     (true,true)    (true,true)    (false,true)     (false,true)   (true,false)
     (true,true)    (true,true)    (false,true)     (true,true)    (true,true) 
     (true,true)    (true,true)    (true,true)      (false,true)   (true,true) 
     (true,true)    (true,true)    (true,true)      (true,true)    (false,true)
     (false,true)   (false,true)   (true,true)   …  (true,true)    (true,true) 
     (false,false)  (false,false)  (true,true)      (true,true)    (true,true) 
     (true,true)    (true,true)    (true,true)      (true,true)    (false,true)
     (true,true)    (true,true)    (false,true)     (true,true)    (true,true) 
     (true,true)    (true,true)    (true,true)      (false,true)   (true,true) 
     (true,true)    (true,true)    (false,true)  …  (true,true)    (true,true) 
     (false,true)   (false,true)   (true,true)      (true,true)    (false,true)
     (true,true)    (true,true)    (true,true)      (true,true)    (false,true)
     ⋮                                           ⋱                             
     (false,true)   (false,true)   (true,true)      (false,true)   (false,true)
     (false,true)   (false,true)   (false,true)     (false,true)   (true,true) 
     (true,true)    (true,true)    (false,true)  …  (false,true)   (true,true) 
     (false,true)   (false,true)   (true,true)      (true,true)    (false,true)
     (true,true)    (true,true)    (true,true)      (false,true)   (true,true) 
     (true,true)    (true,true)    (true,false)     (false,false)  (false,true)
     (true,true)    (true,true)    (true,true)      (true,true)    (false,true)
     (true,true)    (true,true)    (false,true)  …  (true,true)    (true,true) 
     (true,true)    (true,true)    (true,true)      (false,true)   (true,true) 
     (true,true)    (true,true)    (true,true)      (true,true)    (false,true)
     (false,true)   (false,true)   (true,true)      (true,true)    (true,true) 
     (true,true)    (true,true)    (true,true)      (true,true)    (true,true) 



## Summary statistics of SNP data


```julia
people, snps = size(cg10k)
```




    (6670,630860)




```julia
# summary statistics (~50 secs on my laptop)
@time maf, _, missings_by_snp, = summarize(cg10k);
```

     26.436140 seconds (33.44 k allocations: 10.920 MB)



```julia
# 5 number summary and average MAF (minor allele frequencies)
quantile(maf, [0.0 .25 .5 .75 1.0]), mean(maf)
```




    (
    [0.00841726 0.124063 … 0.364253 0.5],
    
    0.24536516625042462)




```julia
# Pkg.add("Plots")
# Pkg.add("PyPlot")
using Plots
pyplot()

histogram(maf, xlab = "Minor Allele Frequency (MAF)", label = "MAF")
```




<img src="data:image/png;base64,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efPmqlKlSrGfw80IDw+Xr6+vjh07pkcffTTf5dq2bavY2FhduHDBHiZct26dMjIyXJarX7++tm7dKsuy7DCX82OHHJ07d1Z2drbOnj1rHwEqaf1xcXFq3bp1vsvVq1dPPXr00LRp07Rz584izSHVtGlTnThxotDpLEaNGqUzZ85ozJgxefZblqWEhASFh4dr6tSpLn1XrlxR7969FR8fX+yAlZKSokceeaRY6wCljYAF3AJt27a1Z582LSYmRn379tXw4cM1fPhw7du3Ty+++KIGDBigFi1aFHt78fHxCg4O1ujRo/M8yrNgwQJ9+umn+X6J3ujhhx9W79691b17dz333HNq2bKlfv31V+3evVsHDx7M9Uu5HOPHj1d8fLw6dOigZ555RkFBQfrll1+0c+dOBQYGuoRJUwICAhQTE6Px48crNTVVHTp0UIUKFXTo0CGtWLFCK1eulNPp1LPPPqvZs2erR48eeu6553TmzBnFxMTkCpwDBw7UggULNG7cOPXu3Vvbtm3T8uXLXZa5++67NXbsWA0bNkx/+9vf1LZtW129elX79u3T1q1btWzZsiLXX6NGDb388suaPHmysrKy9Oijj+r69evauHGjHn/8cbVq1cpedtSoUerTp48CAgJyTSWRl/DwcGVlZemHH34ocH6x++67r8DP+bZt25SamqqpU6e6zA2Xo0ePHlq8eLH+/ve/F1pTjoyMDB04cEAPPPBAkdcB3IGT3IEy5tFHH9Wnn36q//znP+rTp4/+/ve/a/To0Vq4cGGxt3Xy5Elt3LhRkZGReQ4ntmzZUmFhYcWaOXvp0qUaO3asZs2apR49emjEiBFat26dPRt3XmrWrKkdO3YoLCxMkyZNUteuXfXss88qJSXllk52OmnSJM2dO1cbNmxQv379NGjQIM2dO1ft2rWzzwerX7++Vq9erYsXL2rQoEF6++23NXv2bNWtW9dlW7169dK0adO0fPly9e3bV3v37tXs2bNzPeasWbMUExOjRYsWqWfPnoqMjNTSpUvzDCCFefHFF/XJJ59o27Zt6tu3r5588kkdOnRItWvXdlmuR48ecjqdGjZsWJGGXO+++241b968wKk1imLRokXy8/NT//798+x/4oknlJqaqq1btxZ5m2vWrJGfn5+6detWotqAW81hWXlcUAoAUKDQ0FC1a9cu36NynmTdunXq1q2bkpOT9cc//rFI67z77ruaM2eO9u3bd4urK55+/fqpdu3a+vjjj91dClAgjmABQDl1/Phxbd68WZMmTVKHDh2KHK4kacyYMTp//ny+P1hwh4MHD2rt2rV68cUX3V0KUCgCFgCUU7NmzVLnzp3l5eVV7CM+lStX1oIFC1wuweRux48f19y5cxUSEuLuUoBC/X+9RPK+WRk5DgAAAABJRU5ErkJggg==" />




```julia
# proportion of missing genotypes
sum(missings_by_snp) / length(cg10k)
```




    0.0013128198764010824




```julia
# proportion of rare SNPs with maf < 0.05
countnz(maf .< 0.05) / length(maf)
```




    0.07228069619249913



## Empirical kinship matrix

We estimate empirical kinship based on all SNPs by the genetic relation matrix (GRM). Missing genotypes are imputed on the fly by drawing according to the minor allele frequencies.


```julia
# GRM using all SNPs (~10 mins on my laptop)
srand(123)
@time Φgrm = grm(cg10k; method = :GRM)
```

    2099.911248 seconds (29.50 G allocations: 441.429 GB, 1.50% gc time)





    6670×6670 Array{Float64,2}:
      0.502916      0.00329978   -0.000116213  …  -6.46286e-5   -0.00281229 
      0.00329978    0.49892      -0.00201992       0.000909871   0.00345573 
     -0.000116213  -0.00201992    0.493632         0.000294565  -0.000349854
      0.000933977  -0.00320391   -0.0018611       -0.00241682   -0.00127078 
     -7.75429e-5   -0.0036075     0.00181442       0.00213976   -0.00158382 
      0.00200371    0.000577386   0.0025455    …   0.000943753  -1.82994e-6 
      0.000558503   0.00241421   -0.0018782        0.001217     -0.00123924 
     -0.000659495   0.00319987   -0.00101496       0.00353646   -0.00024093 
     -0.00102619   -0.00120448   -0.00055462       0.00175586    0.00181899 
     -0.00136838    0.00211996    0.000119128     -0.00147305   -0.00105239 
     -0.00206144    0.000148818  -0.000475177  …  -0.000265522  -0.00106123 
      0.000951016   0.00167042    0.00183545      -0.000703658  -0.00313334 
      0.000330442  -0.000904147   0.00301478       0.000754772  -0.00127413 
      ⋮                                        ⋱                            
      0.00301137    0.00116042    0.00100426       6.67254e-6    0.00307069 
     -0.00214008    0.00270925   -0.00185054      -0.00109935    0.00366816 
      0.000546739  -0.00242646   -0.00305264   …  -0.000629014   0.00210779 
     -0.00422553   -0.0020713    -0.00109052      -0.000705804  -0.000508055
     -0.00318405   -0.00075385    0.00312377       0.00052883   -3.60969e-5 
      0.000430196  -0.00197163    0.00268545      -0.00633175   -0.00520337 
      0.00221429    0.000849792  -0.00101111      -0.000943129  -0.000624419
     -0.00229025   -0.000130598   0.000101853  …   0.000840136  -0.00230224 
     -0.00202917    0.00233007   -0.00131006       0.00197798   -0.000513771
     -0.000964907  -0.000872326  -7.06722e-5       0.00124702   -0.00295844 
     -6.46286e-5    0.000909871   0.000294565      0.500983      0.000525615
     -0.00281229    0.00345573   -0.000349854      0.000525615   0.500792   



## Phenotypes

Read in the phenotype data and compute descriptive statistics.


```julia
# Pkg.add("DataFrames")
using DataFrames

cg10k_trait = readtable(
    "cg10k_traits.txt"; 
    separator = ' ',
    names = [:FID; :IID; :Trait1; :Trait2; :Trait3; :Trait4; :Trait5; :Trait6; 
             :Trait7; :Trait8; :Trait9; :Trait10; :Trait11; :Trait12; :Trait13],  
    eltypes = [String; String; Float64; Float64; Float64; Float64; Float64; 
               Float64; Float64; Float64; Float64; Float64; Float64; Float64; Float64]
    )
# do not display FID and IID for privacy
cg10k_trait[:, 3:end]
```




<table class="data-frame"><thead><tr><th></th><th>Trait1</th><th>Trait2</th><th>Trait3</th><th>Trait4</th><th>Trait5</th><th>Trait6</th><th>Trait7</th><th>Trait8</th><th>Trait9</th><th>Trait10</th><th>Trait11</th><th>Trait12</th><th>Trait13</th></tr></thead><tbody><tr><th>1</th><td>-1.81573145026234</td><td>-0.94615046147283</td><td>1.11363077580442</td><td>-2.09867121119159</td><td>0.744416614111748</td><td>0.00139171884080131</td><td>0.934732480409667</td><td>-1.22677315418103</td><td>1.1160784277875</td><td>-0.4436280335029</td><td>0.824465656443384</td><td>-1.02852542216546</td><td>-0.394049201727681</td></tr><tr><th>2</th><td>-1.24440094378729</td><td>0.109659992547179</td><td>0.467119394241789</td><td>-1.62131304097589</td><td>1.0566758355683</td><td>0.978946979419181</td><td>1.00014633946047</td><td>0.32487427140228</td><td>1.16232175219696</td><td>2.6922706948705</td><td>3.08263672461047</td><td>1.09064954786013</td><td>0.0256616415357438</td></tr><tr><th>3</th><td>1.45566914502305</td><td>1.53866932923243</td><td>1.09402959376555</td><td>0.586655272226893</td><td>-0.32796454430367</td><td>-0.30337709778827</td><td>-0.0334354881314741</td><td>-0.464463064285437</td><td>-0.3319396273436</td><td>-0.486839089635991</td><td>-1.10648681564373</td><td>-1.42015780427231</td><td>-0.687463456644413</td></tr><tr><th>4</th><td>-0.768809276698548</td><td>0.513490885514249</td><td>0.244263028382142</td><td>-1.31740254475691</td><td>1.19393774326845</td><td>1.17344127734288</td><td>1.08737426675232</td><td>0.536022583732261</td><td>0.802759240762068</td><td>0.234159411749815</td><td>0.394174866891074</td><td>-0.767365892476029</td><td>0.0635385761884935</td></tr><tr><th>5</th><td>-0.264415132547719</td><td>-0.348240421825694</td><td>-0.0239065083413606</td><td>0.00473915802244948</td><td>1.25619191712193</td><td>1.2038883667631</td><td>1.29800739042627</td><td>0.310113660247311</td><td>0.626159861059352</td><td>0.899289129831224</td><td>0.54996783350812</td><td>0.540687809542048</td><td>0.179675416046033</td></tr><tr><th>6</th><td>-1.37617270917293</td><td>-1.47191967744564</td><td>0.291179894254146</td><td>-0.803110740704731</td><td>-0.264239977442213</td><td>-0.260573027836772</td><td>-0.165372266287781</td><td>-0.219257294118362</td><td>1.04702422290318</td><td>-0.0985815534616482</td><td>0.947393438068448</td><td>0.594014812031438</td><td>0.245407436348479</td></tr><tr><th>7</th><td>0.1009416296374</td><td>-0.191615722103455</td><td>-0.567421321596677</td><td>0.378571487240382</td><td>-0.246656179817904</td><td>-0.608810750053858</td><td>0.189081058215596</td><td>-1.27077787326519</td><td>-0.452476199143965</td><td>0.702562877297724</td><td>0.332636218957179</td><td>0.0026916503626181</td><td>0.317117176705358</td></tr><tr><th>8</th><td>-0.319818276367464</td><td>1.35774480657283</td><td>0.818689545938528</td><td>-1.15565531644352</td><td>0.63448368102259</td><td>0.291461908634679</td><td>0.933323714954726</td><td>-0.741083289682492</td><td>0.647477683507572</td><td>-0.970877627077966</td><td>0.220861165411304</td><td>0.852512250237764</td><td>-0.225904624283945</td></tr><tr><th>9</th><td>-0.288334173342032</td><td>0.566082538090752</td><td>0.254958336116175</td><td>-0.652578302869714</td><td>0.668921559277347</td><td>0.978309199170558</td><td>0.122862966041938</td><td>1.4790926378214</td><td>0.0672132424173449</td><td>0.0795903917527827</td><td>0.167532455243232</td><td>0.246915579442139</td><td>0.539932616458363</td></tr><tr><th>10</th><td>-1.15759732583991</td><td>-0.781198583545165</td><td>-0.595807759833517</td><td>-1.00554980260402</td><td>0.789828885933321</td><td>0.571058413379044</td><td>0.951304176233755</td><td>-0.295962982984816</td><td>0.99042002479707</td><td>0.561309366988983</td><td>0.733100030623233</td><td>-1.73467772245684</td><td>-1.35278484330654</td></tr><tr><th>11</th><td>0.740569150459031</td><td>1.40873846755415</td><td>0.734689999440088</td><td>0.0208322841295094</td><td>-0.337440968561619</td><td>-0.458304040611395</td><td>-0.142582512772326</td><td>-0.580392297464107</td><td>-0.684684998101516</td><td>-0.00785381461893456</td><td>-0.712244337518008</td><td>-0.313345561230878</td><td>-0.345419463162219</td></tr><tr><th>12</th><td>-0.675892486454995</td><td>0.279892613829682</td><td>0.267915996308248</td><td>-1.04103665392985</td><td>0.910741715645888</td><td>0.866027618513171</td><td>1.07414431702005</td><td>0.0381751003538302</td><td>0.766355377018601</td><td>-0.340118016143495</td><td>-0.809013958505059</td><td>0.548521663785885</td><td>-0.0201828675962336</td></tr><tr><th>13</th><td>-0.795410435603455</td><td>-0.699989939762738</td><td>0.3991295030063</td><td>-0.510476261900736</td><td>1.51552245416844</td><td>1.28743032939467</td><td>1.53772393250903</td><td>0.133989160117702</td><td>1.02025736886037</td><td>0.499018733899186</td><td>-0.36948273277931</td><td>-1.10153460436318</td><td>-0.598132438886619</td></tr><tr><th>14</th><td>-0.193483122930324</td><td>-0.286021160323518</td><td>-0.691494225262995</td><td>0.0131581678700699</td><td>1.52337470686782</td><td>1.4010638072262</td><td>1.53114620451896</td><td>0.333066483478075</td><td>1.04372480381099</td><td>0.163206783570466</td><td>-0.422883765001728</td><td>-0.383527976713573</td><td>-0.489221907788158</td></tr><tr><th>15</th><td>0.151246203379718</td><td>2.09185108993614</td><td>2.03800472474384</td><td>-1.12474717143531</td><td>1.66557024390713</td><td>1.62535675109576</td><td>1.58751070483655</td><td>0.635852186043776</td><td>0.842577784605979</td><td>0.450761870778952</td><td>-1.39479033623028</td><td>-0.560984107567768</td><td>0.289349776549287</td></tr><tr><th>16</th><td>-0.464608740812712</td><td>0.36127694772303</td><td>1.2327673928287</td><td>-0.826033731086383</td><td>1.43475224709983</td><td>1.74451823818846</td><td>0.211096887484638</td><td>2.64816425140548</td><td>1.02511433146096</td><td>0.11975731603184</td><td>0.0596832073448267</td><td>-0.631231612661616</td><td>-0.207878671782927</td></tr><tr><th>17</th><td>-0.732977488012215</td><td>-0.526223425889779</td><td>0.61657871336593</td><td>-0.55447974332593</td><td>0.947484859025104</td><td>0.936833214138173</td><td>0.972516806335524</td><td>0.290251013865227</td><td>1.01285359725723</td><td>0.516207422283291</td><td>-0.0300689171988194</td><td>0.8787322524583</td><td>0.450254629309513</td></tr><tr><th>18</th><td>-0.167326459622119</td><td>0.175327165487237</td><td>0.287467725892572</td><td>-0.402652532084246</td><td>0.551181509418056</td><td>0.522204743290975</td><td>0.436837660094653</td><td>0.299564933845579</td><td>0.583109520896067</td><td>-0.704415820005353</td><td>-0.730810367994577</td><td>-1.95140580379896</td><td>-0.933504665700164</td></tr><tr><th>19</th><td>1.41159485787418</td><td>1.78722407901017</td><td>0.84397639585364</td><td>0.481278083772991</td><td>-0.0887673728508268</td><td>-0.49957757426858</td><td>0.304195897924847</td><td>-1.23884208383369</td><td>-0.153475724036624</td><td>-0.870486102788329</td><td>0.0955473331150403</td><td>-0.983708050882817</td><td>-0.3563445644514</td></tr><tr><th>20</th><td>-1.42997091652825</td><td>-0.490147045034213</td><td>0.272730237607695</td><td>-1.61029992954153</td><td>0.990787817197748</td><td>0.711687532608184</td><td>1.1885836012715</td><td>-0.371229188075638</td><td>1.24703459239952</td><td>-0.0389162332271516</td><td>0.883495749072872</td><td>2.58988026321017</td><td>3.33539552370368</td></tr><tr><th>21</th><td>-0.147247288176765</td><td>0.12328430415652</td><td>0.617549051912237</td><td>-0.18713077178262</td><td>0.256438107586694</td><td>0.17794983735083</td><td>0.412611806463263</td><td>-0.244809124559737</td><td>0.0947624806136492</td><td>0.723017223849532</td><td>-0.683948354633436</td><td>0.0873751276309269</td><td>-0.262209652750371</td></tr><tr><th>22</th><td>-0.187112676773894</td><td>-0.270777264595619</td><td>-1.01556818551606</td><td>0.0602850568600233</td><td>0.272419757757978</td><td>0.869133161879197</td><td>-0.657519461414234</td><td>2.32388522018189</td><td>-0.999936011525034</td><td>1.44671844178306</td><td>0.971157886040772</td><td>-0.358747904241515</td><td>-0.439657942096136</td></tr><tr><th>23</th><td>-1.82434047163768</td><td>-0.933480446068067</td><td>1.29474003766977</td><td>-1.94545221151036</td><td>0.33584651189654</td><td>0.359201654302844</td><td>0.513652924365886</td><td>-0.073197696696958</td><td>1.57139042812005</td><td>1.53329371326728</td><td>1.82076821859528</td><td>2.22740301867829</td><td>1.50063347195857</td></tr><tr><th>24</th><td>-2.29344084351335</td><td>-2.49161842344418</td><td>0.40383988742336</td><td>-2.36488074752948</td><td>1.4105254831956</td><td>1.42244117147792</td><td>1.17024166272172</td><td>0.84476650176855</td><td>1.79026875432495</td><td>0.648181858970515</td><td>-0.0857231057403538</td><td>-1.02789535292617</td><td>0.491288088952859</td></tr><tr><th>25</th><td>-0.434135932888305</td><td>0.740881989034652</td><td>0.699576357578518</td><td>-1.02405543187775</td><td>0.759529223983713</td><td>0.956656110895288</td><td>0.633299568656589</td><td>0.770733932268516</td><td>0.824988511714526</td><td>1.84287437634769</td><td>1.91045942063443</td><td>-0.502317207869366</td><td>0.132670133448219</td></tr><tr><th>26</th><td>-2.1920969546557</td><td>-2.49465664272271</td><td>0.354854763893431</td><td>-1.93155848635714</td><td>0.941979400289938</td><td>0.978917101414106</td><td>0.894860097289736</td><td>0.463239402831873</td><td>1.12537133317163</td><td>1.70528446191955</td><td>0.717792714479123</td><td>0.645888049108261</td><td>0.783968250169388</td></tr><tr><th>27</th><td>-1.46602269088422</td><td>-1.24921677101897</td><td>0.307977693653039</td><td>-1.55097364660989</td><td>0.618908494474798</td><td>0.662508171662042</td><td>0.475957173906078</td><td>0.484718674597707</td><td>0.401564892028249</td><td>0.55987973254026</td><td>-0.376938143754217</td><td>-0.933982629228218</td><td>0.390013151672955</td></tr><tr><th>28</th><td>-1.83317744236881</td><td>-1.53268787828701</td><td>2.55674262685865</td><td>-1.51827745783835</td><td>0.789409601746455</td><td>0.908747799728588</td><td>0.649971922941479</td><td>0.668373649931667</td><td>1.20058303519903</td><td>0.277963256075637</td><td>1.2504953198275</td><td>3.31370445071638</td><td>2.22035828885342</td></tr><tr><th>29</th><td>-0.784546628243178</td><td>0.276582579543931</td><td>3.01104958800057</td><td>-1.11978843206758</td><td>0.920823858422707</td><td>0.750217689886151</td><td>1.26153730009639</td><td>-0.403363882922417</td><td>0.400667296857811</td><td>-0.217597941303479</td><td>-0.724669537565068</td><td>-0.391945338467193</td><td>-0.650023936358253</td></tr><tr><th>30</th><td>0.464455916345135</td><td>1.3326356122229</td><td>-1.23059563374303</td><td>-0.357975958937414</td><td>1.18249746977104</td><td>1.54315938069757</td><td>-0.60339041154062</td><td>3.38308845958422</td><td>0.823740765148641</td><td>-0.129951318508883</td><td>-0.657979878422938</td><td>-0.499534924074273</td><td>-0.414476569095651</td></tr><tr><th>&vellip;</th><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td><td>&vellip;</td></tr></tbody></table>




```julia
describe(cg10k_trait[:, 3:end])
```

    Trait1
    Min      -3.2041280147118
    1st Qu.  -0.645770976594801
    Median   0.12500996951180798
    Mean     0.0022113846331389903
    3rd Qu.  0.7233154897636109
    Max      3.47939787136478
    NAs      0
    NA%      0.0%
    
    Trait2
    Min      -3.51165862877157
    1st Qu.  -0.6426205239938769
    Median   0.0335172506981786
    Mean     0.0013525291443179934
    3rd Qu.  0.6574666174104795
    Max      4.91342267449592
    NAs      0
    NA%      0.0%
    
    Trait3
    Min      -3.93843646263987
    1st Qu.  -0.6409067201835312
    Median   -0.000782161570259152
    Mean     -0.0012959062525954158
    3rd Qu.  0.6371084235689337
    Max      7.91629946619107
    NAs      0
    NA%      0.0%
    
    Trait4
    Min      -3.60840330795393
    1st Qu.  -0.5460856267376792
    Median   0.228165419346029
    Mean     0.0023089259432067487
    3rd Qu.  0.7152907338009037
    Max      3.12768818152017
    NAs      0
    NA%      0.0%
    
    Trait5
    Min      -4.14874907974159
    1st Qu.  -0.6907651815712424
    Median   0.03103429560265845
    Mean     -0.0017903947913742396
    3rd Qu.  0.7349158784775832
    Max      2.71718436484651
    NAs      0
    NA%      0.0%
    
    Trait6
    Min      -3.82479174931095
    1st Qu.  -0.66279630694076
    Median   0.03624198629403995
    Mean     -0.001195980597331062
    3rd Qu.  0.7411755243742835
    Max      2.58972802240228
    NAs      0
    NA%      0.0%
    
    Trait7
    Min      -4.27245540828955
    1st Qu.  -0.6389232588654877
    Median   0.0698010018021233
    Mean     -0.0019890555724116853
    3rd Qu.  0.7104228734967848
    Max      2.65377857124275
    NAs      0
    NA%      0.0%
    
    Trait8
    Min      -5.62548796912517
    1st Qu.  -0.6015747053036895
    Median   -0.0386301401797661
    Mean     0.0006140754882985941
    3rd Qu.  0.5273417705306229
    Max      5.8057022359485
    NAs      0
    NA%      0.0%
    
    Trait9
    Min      -5.38196778211456
    1st Qu.  -0.6014287731518225
    Median   0.10657100636146799
    Mean     -0.0018096522573535152
    3rd Qu.  0.6985667613073132
    Max      2.57193558386964
    NAs      0
    NA%      0.0%
    
    Trait10
    Min      -3.54850550601412
    1st Qu.  -0.6336406665339003
    Median   -0.0966507436079331
    Mean     -0.0004370294352533275
    3rd Qu.  0.498610364573902
    Max      6.53782005410551
    NAs      0
    NA%      0.0%
    
    Trait11
    Min      -3.26491021902041
    1st Qu.  -0.6736846396608624
    Median   -0.0680437088585371
    Mean     -0.0006159181111862523
    3rd Qu.  0.6554864114250585
    Max      4.26240968462615
    NAs      0
    NA%      0.0%
    
    Trait12
    Min      -8.85190902714652
    1st Qu.  -0.5396855871831098
    Median   -0.1410985990171995
    Mean     -0.0005887830910961934
    3rd Qu.  0.35077884186653374
    Max      13.2114017261714
    NAs      0
    NA%      0.0%
    
    Trait13
    Min      -5.59210353493304
    1st Qu.  -0.49228904714392474
    Median   -0.14102175804213551
    Mean     -0.0001512383146454028
    3rd Qu.  0.32480412746681697
    Max      24.174436145414
    NAs      0
    NA%      0.0%
    



```julia
Y = convert(Matrix{Float64}, cg10k_trait[:, 3:15])
histogram(Y, layout = 13)
```




<img src="data:image/png;base64,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" />



## Pre-processing data for heritability analysis

To prepare variance component model fitting, we form an instance of `VarianceComponentVariate`. The two variance components are $(2\Phi, I)$.


```julia
using VarianceComponentModels

# form data as VarianceComponentVariate
cg10kdata = VarianceComponentVariate(Y, (2Φgrm, eye(size(Y, 1))))
fieldnames(cg10kdata)
```




    3-element Array{Symbol,1}:
     :Y
     :X
     :V




```julia
cg10kdata
```




    VarianceComponentModels.VarianceComponentVariate{Float64,2,Array{Float64,2},Array{Float64,2},Array{Float64,2}}([-1.81573 -0.94615 … -1.02853 -0.394049; -1.2444 0.10966 … 1.09065 0.0256616; … ; 0.886626 0.487408 … -0.636874 -0.439825; -1.24394 0.213697 … 0.299931 0.392809],,(
    [1.00583 0.00659955 … -0.000129257 -0.00562459; 0.00659955 0.99784 … 0.00181974 0.00691145; … ; -0.000129257 0.00181974 … 1.00197 0.00105123; -0.00562459 0.00691145 … 0.00105123 1.00158],
    
    [1.0 0.0 … 0.0 0.0; 0.0 1.0 … 0.0 0.0; … ; 0.0 0.0 … 1.0 0.0; 0.0 0.0 … 0.0 1.0]))



Before fitting the variance component model, we pre-compute the eigen-decomposition of $2\Phi_{\text{GRM}}$, the rotated responses, and the constant part in log-likelihood, and store them as a `TwoVarCompVariateRotate` instance, which is re-used in various variane component estimation procedures.


```julia
# pre-compute eigen-decomposition (~50 secs on my laptop)
@time cg10kdata_rotated = TwoVarCompVariateRotate(cg10kdata)
fieldnames(cg10kdata_rotated)
```

     46.051433 seconds (726.58 k allocations: 1.024 GB, 0.45% gc time)





    5-element Array{Symbol,1}:
     :Yrot    
     :Xrot    
     :eigval  
     :eigvec  
     :logdetV2



## Save intermediate results

We don't want to re-compute SnpArray and empirical kinship matrices again and again for heritibility analysis.


```julia
# Pkg.add("JLD")
#using JLD
#@save "cg10k.jld"
#whos()
```

                              Base  47039 KB     Module
                           BinDeps    218 KB     Module
                             Blosc     59 KB     Module
                        ColorTypes   8492 KB     Module
                            Colors   8414 KB     Module
                            Compat   8100 KB     Module
                             Conda   8491 KB     Module
                              Core  19714 KB     Module
                        DataArrays   8444 KB     Module
                        DataFrames   8940 KB     Module
                    DataStructures    234 KB     Module
                            FileIO   9044 KB     Module
                 FixedPointNumbers   8573 KB     Module
                   FixedSizeArrays   8222 KB     Module
                              GZip   8003 KB     Module
                              HDF5   8809 KB     Module
                            IJulia 2082578 KB     Module
                             Ipopt     32 KB     Module
                  IterativeSolvers    333 KB     Module
                         Iterators     44 KB     Module
                               JLD   9092 KB     Module
                              JSON   8124 KB     Module
                            KNITRO    218 KB     Module
                      LaTeXStrings   4622 bytes  Module
                     LegacyStrings     67 KB     Module
                        MacroTools   8190 KB     Module
                              Main 2498588 KB     Module
                      MathProgBase    272 KB     Module
                          Measures     24 KB     Module
                            Nettle   8048 KB     Module
                        PlotThemes   7999 KB     Module
                         PlotUtils   8264 KB     Module
                             Plots  12383 KB     Module
                            PyCall  10435 KB     Module
                            PyPlot   9938 KB     Module
                       RecipesBase   8158 KB     Module
                          Reexport   6178 bytes  Module
                               SHA     71 KB     Module
                           Showoff   8024 KB     Module
                         SnpArrays   8191 KB     Module
                 SortingAlgorithms     28 KB     Module
                  SpecialFunctions   8257 KB     Module
                         StatsBase   8662 KB     Module
                         URIParser   8064 KB     Module
           VarianceComponentModels    189 KB     Module
                                 Y    677 KB     6670×13 Array{Float64,2}
                               ZMQ   8058 KB     Module
                                 _     77 KB     630860-element BitArray{1}
                             cg10k 1027303 KB     6670×630860 SnpArrays.SnpArray{2}
                       cg10k_trait    978 KB     6670×15 DataFrames.DataFrame
                         cg10kdata 695816 KB     VarianceComponentModels.VarianceCo…
                 cg10kdata_rotated 348299 KB     VarianceComponentModels.TwoVarComp…
                               maf   4928 KB     630860-element Array{Float64,1}
                   missings_by_snp   4928 KB     630860-element Array{Int64,1}
                            people      8 bytes  Int64
                              snps      8 bytes  Int64
                              Φgrm 347569 KB     6670×6670 Array{Float64,2}


To load workspace


```julia
using SnpArrays, JLD, DataFrames, VarianceComponentModels, Plots
pyplot()
@load "cg10k.jld"
whos()
```

                              Base  39406 KB     Module
                           BinDeps    218 KB     Module
                             Blosc     59 KB     Module
                        ColorTypes   6371 KB     Module
                            Colors   6378 KB     Module
                            Compat   6099 KB     Module
                             Conda   6460 KB     Module
                              Core  15532 KB     Module
                        DataArrays   6388 KB     Module
                        DataFrames    649 KB     Module
                    DataStructures    234 KB     Module
                            FileIO   6870 KB     Module
                 FixedPointNumbers   6579 KB     Module
                   FixedSizeArrays   6226 KB     Module
                              GZip   6004 KB     Module
                              HDF5   6657 KB     Module
                            IJulia   7125 KB     Module
                             Ipopt     32 KB     Module
                  IterativeSolvers    333 KB     Module
                         Iterators     44 KB     Module
                               JLD   6839 KB     Module
                              JSON   6132 KB     Module
                            KNITRO    218 KB     Module
                      LaTeXStrings   4622 bytes  Module
                     LegacyStrings     68 KB     Module
                        MacroTools   6196 KB     Module
                              Main 2489170 KB     Module
                      MathProgBase    272 KB     Module
                          Measures     21 KB     Module
                            Nettle   6068 KB     Module
                        PlotThemes   6016 KB     Module
                         PlotUtils   6174 KB     Module
                             Plots   9209 KB     Module
                            PyCall   7377 KB     Module
                            PyPlot   7686 KB     Module
                       RecipesBase    179 KB     Module
                          Reexport   6178 bytes  Module
                               SHA     71 KB     Module
                           Showoff     27 KB     Module
                         SnpArrays     98 KB     Module
                 SortingAlgorithms     28 KB     Module
                  SpecialFunctions   6277 KB     Module
                         StatsBase    573 KB     Module
                         URIParser   6085 KB     Module
           VarianceComponentModels    189 KB     Module
                                 Y    677 KB     6670×13 Array{Float64,2}
                               ZMQ   6076 KB     Module
                             cg10k 1027303 KB     6670×630860 SnpArrays.SnpArray{2}
                       cg10k_trait    978 KB     6670×15 DataFrames.DataFrame
                         cg10kdata 695816 KB     VarianceComponentModels.VarianceCo…
                 cg10kdata_rotated 348299 KB     VarianceComponentModels.TwoVarComp…
                               maf   4928 KB     630860-element Array{Float64,1}
                   missings_by_snp   4928 KB     630860-element Array{Int64,1}
                            people      8 bytes  Int64
                              snps      8 bytes  Int64
                              Φgrm 347569 KB     6670×6670 Array{Float64,2}


## Heritability of single traits

We use Fisher scoring algorithm to fit variance component model for each single trait.


```julia
# heritability from single trait analysis
hST = zeros(13)
# standard errors of estimated heritability
hST_se = zeros(13)
# additive genetic effects
σ2a = zeros(13)
# enviromental effects
σ2e = zeros(13)

tic()
for trait in 1:13
    println(names(cg10k_trait)[trait + 2])
    # form data set for trait j
    traitj_data = TwoVarCompVariateRotate(cg10kdata_rotated.Yrot[:, trait], cg10kdata_rotated.Xrot, 
        cg10kdata_rotated.eigval, cg10kdata_rotated.eigvec, cg10kdata_rotated.logdetV2)
    # initialize model parameters
    traitj_model = VarianceComponentModel(traitj_data)
    # estimate variance components
    _, _, _, Σcov, _, _ = mle_fs!(traitj_model, traitj_data; solver=:Ipopt, verbose=false)
    σ2a[trait] = traitj_model.Σ[1][1]
    σ2e[trait] = traitj_model.Σ[2][1]
    @show σ2a[trait], σ2e[trait]
    h, hse = heritability(traitj_model.Σ, Σcov)
    hST[trait] = h[1]
    hST_se[trait] = hse[1]
end
toc()
```

    Trait1
    (σ2a[trait],σ2e[trait]) = (0.26104123217397623,0.7356884432614108)
    Trait2
    (σ2a[trait],σ2e[trait]) = (0.18874147380283665,0.8106899991616688)
    Trait3
    (σ2a[trait],σ2e[trait]) = (0.3185719276547346,0.6801458862875847)
    Trait4
    (σ2a[trait],σ2e[trait]) = (0.26556901333953487,0.7303588364945325)
    Trait5
    (σ2a[trait],σ2e[trait]) = (0.28123321193922013,0.7167989047155017)
    Trait6
    (σ2a[trait],σ2e[trait]) = (0.2829461149704479,0.7165629534396428)
    Trait7
    (σ2a[trait],σ2e[trait]) = (0.21543856403949083,0.7816211121585646)
    Trait8
    (σ2a[trait],σ2e[trait]) = (0.19412648732666096,0.8055277649986139)
    Trait9
    (σ2a[trait],σ2e[trait]) = (0.24789561127296741,0.7504615853619878)
    Trait10
    (σ2a[trait],σ2e[trait]) = (0.10007455815561886,0.899815277360586)
    Trait11
    (σ2a[trait],σ2e[trait]) = (0.16486778169300415,0.8338002257315682)
    Trait12
    (σ2a[trait],σ2e[trait]) = (0.08298660416198149,0.9158035668415443)
    Trait13
    (σ2a[trait],σ2e[trait]) = (0.05684248094794614,0.9423653381325947)
    elapsed time: 0.19565668 seconds





    0.19565668




```julia
# heritability and standard errors
[hST'; hST_se']
```




    2×13 Array{Float64,2}:
     0.261898  0.188849   0.318981   …  0.165088   0.0830871  0.0568875
     0.079869  0.0867203  0.0741462     0.0887725  0.0944835  0.0953863



## Pairwise traits

Joint analysis of multiple traits is subject to intensive research recently. Following code snippet does joint analysis of all pairs of traits, a total of 78 bivariate variane component models.


```julia
# additive genetic effects (2x2 psd matrices) from bavariate trait analysis;
Σa = Array{Matrix{Float64}}(13, 13)
# environmental effects (2x2 psd matrices) from bavariate trait analysis;
Σe = Array{Matrix{Float64}}(13, 13)

tic()
for i in 1:13
    for j in (i+1):13
        println(names(cg10k_trait)[i + 2], names(cg10k_trait)[j + 2])
        # form data set for (trait1, trait2)
        traitij_data = TwoVarCompVariateRotate(cg10kdata_rotated.Yrot[:, [i;j]], cg10kdata_rotated.Xrot, 
            cg10kdata_rotated.eigval, cg10kdata_rotated.eigvec, cg10kdata_rotated.logdetV2)
        # initialize model parameters
        traitij_model = VarianceComponentModel(traitij_data)
        # estimate variance components
        mle_fs!(traitij_model, traitij_data; solver=:Ipopt, verbose=false)
        Σa[i, j] = traitij_model.Σ[1]
        Σe[i, j] = traitij_model.Σ[2]
        @show Σa[i, j], Σe[i, j]
    end
end
toc()
```

    Trait1Trait2
    (Σa[i,j],Σe[i,j]) = (
    [0.260119 0.176216; 0.176216 0.187376],
    
    [0.736589 0.583892; 0.583892 0.812033])
    Trait1Trait3
    (Σa[i,j],Σe[i,j]) = (
    [0.261564 -0.0131268; -0.0131268 0.319057],
    
    [0.73518 -0.121127; -0.121127 0.679679])
    Trait1Trait4
    (Σa[i,j],Σe[i,j]) = (
    [0.26088 0.222614; 0.222614 0.265581],
    
    [0.735846 0.599435; 0.599435 0.730347])
    Trait1Trait5
    (Σa[i,j],Σe[i,j]) = (
    [0.260783 -0.147012; -0.147012 0.281877],
    
    [0.735937 -0.254584; -0.254584 0.716176])
    Trait1Trait6
    (Σa[i,j],Σe[i,j]) = (
    [0.260707 -0.129356; -0.129356 0.283188],
    
    [0.736013 -0.231361; -0.231361 0.716329])
    Trait1Trait7
    (Σa[i,j],Σe[i,j]) = (
    [0.260308 -0.140258; -0.140258 0.215081],
    
    [0.736406 -0.197805; -0.197805 0.781985])
    Trait1Trait8
    (Σa[i,j],Σe[i,j]) = (
    [0.261035 -0.0335296; -0.0335296 0.194143],
    
    [0.735695 -0.126272; -0.126272 0.805512])
    Trait1Trait9
    (Σa[i,j],Σe[i,j]) = (
    [0.263016 -0.204865; -0.204865 0.246796],
    
    [0.733794 -0.30745; -0.30745 0.751544])
    Trait1Trait10
    (Σa[i,j],Σe[i,j]) = (
    [0.260898 -0.0998176; -0.0998176 0.0970233],
    
    [0.735828 -0.303609; -0.303609 0.902853])
    Trait1Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.26074 -0.138983; -0.138983 0.163063],
    
    [0.735982 -0.359175; -0.359175 0.835595])
    Trait1Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.263069 -0.145536; -0.145536 0.0805136],
    
    [0.733781 -0.0416975; -0.0416975 0.918359])
    Trait1Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.262344 -0.108896; -0.108896 0.051294],
    
    [0.73445 -0.113996; -0.113996 0.947942])
    Trait2Trait3
    (Σa[i,j],Σe[i,j]) = (
    [0.189015 0.146157; 0.146157 0.320529],
    
    [0.810418 0.0974992; 0.0974992 0.678271])
    Trait2Trait4
    (Σa[i,j],Σe[i,j]) = (
    [0.188395 0.0752146; 0.0752146 0.265558],
    
    [0.81103 0.220495; 0.220495 0.730369])
    Trait2Trait5
    (Σa[i,j],Σe[i,j]) = (
    [0.188716 -0.011314; -0.011314 0.281247],
    
    [0.810715 -0.0370105; -0.0370105 0.716786])
    Trait2Trait6
    (Σa[i,j],Σe[i,j]) = (
    [0.188774 -0.0031066; -0.0031066 0.283013],
    
    [0.810658 -0.0211827; -0.0211827 0.716499])
    Trait2Trait7
    (Σa[i,j],Σe[i,j]) = (
    [0.188352 -0.0299579; -0.0299579 0.215189],
    
    [0.811072 -0.00136939; -0.00136939 0.781868])
    Trait2Trait8
    (Σa[i,j],Σe[i,j]) = (
    [0.189262 0.0331423; 0.0331423 0.194666],
    
    [0.810182 -0.0326003; -0.0326003 0.805005])
    Trait2Trait9
    (Σa[i,j],Σe[i,j]) = (
    [0.187285 -0.0854146; -0.0854146 0.246719],
    
    [0.812133 -0.0808791; -0.0808791 0.751617])
    Trait2Trait10
    (Σa[i,j],Σe[i,j]) = (
    [0.188965 -0.125319; -0.125319 0.100121],
    
    [0.810498 -0.271071; -0.271071 0.899849])
    Trait2Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.187762 -0.118479; -0.118479 0.166273],
    
    [0.811653 -0.295549; -0.295549 0.832437])
    Trait2Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.188191 -0.0905383; -0.0905383 0.0822634],
    
    [0.811272 0.045422; 0.045422 0.916586])
    Trait2Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.18826 -0.0707041; -0.0707041 0.0547239],
    
    [0.811217 0.0737977; 0.0737977 0.944521])
    Trait3Trait4
    (Σa[i,j],Σe[i,j]) = (
    [0.31852 -0.154339; -0.154339 0.264754],
    
    [0.680196 -0.30344; -0.30344 0.731152])
    Trait3Trait5
    (Σa[i,j],Σe[i,j]) = (
    [0.31897 0.184354; 0.184354 0.2825],
    
    [0.67976 0.336411; 0.336411 0.715567])
    Trait3Trait6
    (Σa[i,j],Σe[i,j]) = (
    [0.319566 0.16664; 0.16664 0.285031],
    
    [0.679183 0.297698; 0.297698 0.714536])
    Trait3Trait7
    (Σa[i,j],Σe[i,j]) = (
    [0.318576 0.166852; 0.166852 0.215232],
    
    [0.680142 0.347139; 0.347139 0.781823])
    Trait3Trait8
    (Σa[i,j],Σe[i,j]) = (
    [0.320499 0.0575319; 0.0575319 0.197245],
    
    [0.678283 0.0442597; 0.0442597 0.802474])
    Trait3Trait9
    (Σa[i,j],Σe[i,j]) = (
    [0.318719 0.137292; 0.137292 0.246976],
    
    [0.680004 0.267105; 0.267105 0.751357])
    Trait3Trait10
    (Σa[i,j],Σe[i,j]) = (
    [0.318915 -0.0786338; -0.0786338 0.101103],
    
    [0.679815 -0.140789; -0.140789 0.898798])
    Trait3Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.317822 -0.017984; -0.017984 0.164743],
    
    [0.680871 -0.114166; -0.114166 0.833923])
    Trait3Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.320888 0.0845248; 0.0845248 0.0869868],
    
    [0.677914 0.0340133; 0.0340133 0.911841])
    Trait3Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.323009 0.110681; 0.110681 0.0611739],
    
    [0.675901 -0.00729662; -0.00729662 0.938072])
    Trait4Trait5
    (Σa[i,j],Σe[i,j]) = (
    [0.265667 -0.215848; -0.215848 0.282919],
    
    [0.730254 -0.376675; -0.376675 0.715164])
    Trait4Trait6
    (Σa[i,j],Σe[i,j]) = (
    [0.266143 -0.200634; -0.200634 0.284442],
    
    [0.729794 -0.346804; -0.346804 0.715112])
    Trait4Trait7
    (Σa[i,j],Σe[i,j]) = (
    [0.26449 -0.182752; -0.182752 0.214117],
    
    [0.731415 -0.326172; -0.326172 0.782935])
    Trait4Trait8
    (Σa[i,j],Σe[i,j]) = (
    [0.266694 -0.0976354; -0.0976354 0.196126],
    
    [0.729266 -0.15036; -0.15036 0.803571])
    Trait4Trait9
    (Σa[i,j],Σe[i,j]) = (
    [0.270037 -0.227407; -0.227407 0.248046],
    
    [0.726025 -0.415601; -0.415601 0.750298])
    Trait4Trait10
    (Σa[i,j],Σe[i,j]) = (
    [0.265543 -0.0338107; -0.0338107 0.0996098],
    
    [0.730395 -0.227725; -0.227725 0.900275])
    Trait4Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.265628 -0.09674; -0.09674 0.163274],
    
    [0.730302 -0.272611; -0.272611 0.835371])
    Trait4Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.268164 -0.141613; -0.141613 0.0803968],
    
    [0.727883 -0.0828465; -0.0828465 0.918446])
    Trait4Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.266171 -0.0980731; -0.0980731 0.0540102],
    
    [0.729775 -0.22506; -0.22506 0.945204])
    Trait5Trait6
    (Σa[i,j],Σe[i,j]) = (
    [0.281592 0.280898; 0.280898 0.282298],
    
    [0.716455 0.660368; 0.660368 0.717195])
    Trait5Trait7
    (Σa[i,j],Σe[i,j]) = (
    [0.280814 0.2321; 0.2321 0.211662],
    
    [0.717218 0.674304; 0.674304 0.785343])
    Trait5Trait8
    (Σa[i,j],Σe[i,j]) = (
    [0.28134 0.163948; 0.163948 0.192703],
    
    [0.716701 0.221032; 0.221032 0.806922])
    Trait5Trait9
    (Σa[i,j],Σe[i,j]) = (
    [0.283878 0.244532; 0.244532 0.241293],
    
    [0.714244 0.508417; 0.508417 0.756894])
    Trait5Trait10
    (Σa[i,j],Σe[i,j]) = (
    [0.281813 -0.0462114; -0.0462114 0.101481],
    
    [0.716238 -0.0572111; -0.0572111 0.898424])
    Trait5Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.280424 0.0202495; 0.0202495 0.164003],
    
    [0.717586 -0.0352436; -0.0352436 0.834649])
    Trait5Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.281427 0.0616131; 0.0616131 0.0827166],
    
    [0.716615 0.0529286; 0.0529286 0.916074])
    Trait5Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.282292 0.0704221; 0.0704221 0.0569463],
    
    [0.715782 0.0528374; 0.0528374 0.942268])
    Trait6Trait7
    (Σa[i,j],Σe[i,j]) = (
    [0.282961 0.220656; 0.220656 0.213856],
    
    [0.716549 0.581083; 0.581083 0.783179])
    Trait6Trait8
    (Σa[i,j],Σe[i,j]) = (
    [0.282961 0.18408; 0.18408 0.192379],
    
    [0.716549 0.436597; 0.436597 0.807246])
    Trait6Trait9
    (Σa[i,j],Σe[i,j]) = (
    [0.284978 0.234436; 0.234436 0.243207],
    
    [0.714601 0.476826; 0.476826 0.755028])
    Trait6Trait10
    (Σa[i,j],Σe[i,j]) = (
    [0.283655 -0.0435485; -0.0435485 0.102025],
    
    [0.715877 -0.0591681; -0.0591681 0.897886])
    Trait6Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.281522 0.0279992; 0.0279992 0.16343],
    
    [0.717946 -0.0524106; -0.0524106 0.835213])
    Trait6Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.283114 0.0571399; 0.0571399 0.0826789],
    
    [0.716403 0.0479199; 0.0479199 0.916112])
    Trait6Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.28382 0.0611212; 0.0611212 0.0570817],
    
    [0.715722 0.0532698; 0.0532698 0.942133])
    Trait7Trait8
    (Σa[i,j],Σe[i,j]) = (
    [0.213857 0.0884555; 0.0884555 0.192373],
    
    [0.783178 -0.0568331; -0.0568331 0.807252])
    Trait7Trait9
    (Σa[i,j],Σe[i,j]) = (
    [0.218756 0.217042; 0.217042 0.244144],
    
    [0.778433 0.462901; 0.462901 0.754123])
    Trait7Trait10
    (Σa[i,j],Σe[i,j]) = (
    [0.216273 -0.0421146; -0.0421146 0.10209],
    
    [0.780807 -0.0859075; -0.0859075 0.897822])
    Trait7Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.214069 0.0206967; 0.0206967 0.163474],
    
    [0.782961 -0.048148; -0.048148 0.83517])
    Trait7Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.214932 0.0757874; 0.0757874 0.0808731],
    
    [0.782132 0.0346956; 0.0346956 0.917915])
    Trait7Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.215959 0.0749373; 0.0749373 0.0545934],
    
    [0.781139 0.0389058; 0.0389058 0.944622])
    Trait8Trait9
    (Σa[i,j],Σe[i,j]) = (
    [0.194555 0.112816; 0.112816 0.247244],
    
    [0.805124 0.184778; 0.184778 0.751098])
    Trait8Trait10
    (Σa[i,j],Σe[i,j]) = (
    [0.194441 -0.0156342; -0.0156342 0.100426],
    
    [0.805222 0.0119826; 0.0119826 0.899468])
    Trait8Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.193853 0.0225325; 0.0225325 0.164669],
    
    [0.805796 -0.0272747; -0.0272747 0.833997])
    Trait8Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.193951 -0.00287601; -0.00287601 0.0828573],
    
    [0.8057 0.0336128; 0.0336128 0.915932])
    Trait8Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.19398 0.00407867; 0.00407867 0.0569071],
    
    [0.805672 0.0378817; 0.0378817 0.942301])
    Trait9Trait10
    (Σa[i,j],Σe[i,j]) = (
    [0.247294 -0.00230836; -0.00230836 0.0998264],
    
    [0.751051 0.0740729; 0.0740729 0.900061])
    Trait9Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.247823 0.0318235; 0.0318235 0.16489],
    
    [0.750532 0.152285; 0.152285 0.833778])
    Trait9Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.250335 0.0845714; 0.0845714 0.0887587],
    
    [0.748091 0.107756; 0.107756 0.910108])
    Trait9Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.249442 0.0934845; 0.0934845 0.057932],
    
    [0.748975 0.0982191; 0.0982191 0.941335])
    Trait10Trait11
    (Σa[i,j],Σe[i,j]) = (
    [0.0931397 0.100034; 0.100034 0.164955],
    
    [0.906703 0.474427; 0.474427 0.833715])
    Trait10Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.0967247 0.0564043; 0.0564043 0.0794528],
    
    [0.90315 0.0853232; 0.0853232 0.919334])
    Trait10Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.100985 -0.0279916; -0.0279916 0.0578369],
    
    [0.898937 0.166051; 0.166051 0.94139])
    Trait11Trait12
    (Σa[i,j],Σe[i,j]) = (
    [0.163841 0.0570318; 0.0570318 0.0792144],
    
    [0.834814 0.145597; 0.145597 0.919552])
    Trait11Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.164883 -0.00158414; -0.00158414 0.0574968],
    
    [0.833798 0.200612; 0.200612 0.941715])
    Trait12Trait13
    (Σa[i,j],Σe[i,j]) = (
    [0.0845946 0.0685052; 0.0685052 0.0554759],
    
    [0.914214 0.573152; 0.573152 0.943735])
    elapsed time: 7.752196517 seconds





    7.752196517



## 3-trait analysis

Researchers want to jointly analyze traits 5-7. Our strategy is to try both Fisher scoring and MM algorithm with different starting point, and choose the best local optimum. We first form the data set and run Fisher scoring, which yields a final objective value -1.4700991+04.


```julia
traitidx = 5:7
# form data set
trait57_data = TwoVarCompVariateRotate(cg10kdata_rotated.Yrot[:, traitidx], cg10kdata_rotated.Xrot, 
    cg10kdata_rotated.eigval, cg10kdata_rotated.eigvec, cg10kdata_rotated.logdetV2)
# initialize model parameters
trait57_model = VarianceComponentModel(trait57_data)
# estimate variance components
@time mle_fs!(trait57_model, trait57_data; solver=:Ipopt, verbose=true)
trait57_model
```

    This is Ipopt version 3.12.4, running with linear solver mumps.
    NOTE: Other linear solvers might be more efficient (see Ipopt documentation).
    
    Number of nonzeros in equality constraint Jacobian...:        0
    Number of nonzeros in inequality constraint Jacobian.:        0
    Number of nonzeros in Lagrangian Hessian.............:       78
    
    Total number of variables............................:       12
                         variables with only lower bounds:        0
                    variables with lower and upper bounds:        0
                         variables with only upper bounds:        0
    Total number of equality constraints.................:        0
    Total number of inequality constraints...............:        0
            inequality constraints with only lower bounds:        0
       inequality constraints with lower and upper bounds:        0
            inequality constraints with only upper bounds:        0
    
    iter    objective    inf_pr   inf_du lg(mu)  ||d||  lg(rg) alpha_du alpha_pr  ls
       0  3.0247512e+04 0.00e+00 1.00e+02   0.0 0.00e+00    -  0.00e+00 0.00e+00   0 
       5  1.6834796e+04 0.00e+00 4.07e+02 -11.0 3.66e-01    -  1.00e+00 1.00e+00f  1 MaxS
      10  1.4744497e+04 0.00e+00 1.12e+02 -11.0 2.45e-01    -  1.00e+00 1.00e+00f  1 MaxS
      15  1.4701497e+04 0.00e+00 1.30e+01 -11.0 1.15e-01  -4.5 1.00e+00 1.00e+00f  1 MaxS
      20  1.4700992e+04 0.00e+00 6.65e-01 -11.0 1.74e-04  -6.9 1.00e+00 1.00e+00f  1 MaxS
      25  1.4700991e+04 0.00e+00 2.77e-02 -11.0 7.36e-06  -9.2 1.00e+00 1.00e+00f  1 MaxS
      30  1.4700991e+04 0.00e+00 1.15e-03 -11.0 3.06e-07 -11.6 1.00e+00 1.00e+00f  1 MaxS
      35  1.4700991e+04 0.00e+00 4.76e-05 -11.0 1.27e-08 -14.0 1.00e+00 1.00e+00h  1 MaxS
      40  1.4700991e+04 0.00e+00 1.97e-06 -11.0 5.26e-10 -16.4 1.00e+00 1.00e+00f  1 MaxSA
      45  1.4700991e+04 0.00e+00 8.17e-08 -11.0 2.18e-11 -18.8 1.00e+00 1.00e+00h  1 MaxSA
    iter    objective    inf_pr   inf_du lg(mu)  ||d||  lg(rg) alpha_du alpha_pr  ls
    
    Number of Iterations....: 49
    
                                       (scaled)                 (unscaled)
    Objective...............:   4.4724330090668286e+02    1.4700991028593347e+04
    Dual infeasibility......:   6.4345872286001950e-09    2.1150637455850896e-07
    Constraint violation....:   0.0000000000000000e+00    0.0000000000000000e+00
    Complementarity.........:   0.0000000000000000e+00    0.0000000000000000e+00
    Overall NLP error.......:   6.4345872286001950e-09    2.1150637455850896e-07
    
    
    Number of objective function evaluations             = 50
    Number of objective gradient evaluations             = 50
    Number of equality constraint evaluations            = 0
    Number of inequality constraint evaluations          = 0
    Number of equality constraint Jacobian evaluations   = 0
    Number of inequality constraint Jacobian evaluations = 0
    Number of Lagrangian Hessian evaluations             = 49
    Total CPU secs in IPOPT (w/o function evaluations)   =      0.013
    Total CPU secs in NLP function evaluations           =      0.085
    
    EXIT: Optimal Solution Found.
      0.118428 seconds (66.82 k allocations: 6.250 MB)





    VarianceComponentModels.VarianceComponentModel{Float64,2,Array{Float64,2},Array{Float64,2}}(,(
    [0.281163 0.280014 0.232384; 0.280014 0.284899 0.220285; 0.232384 0.220285 0.212687],
    
    [0.716875 0.66125 0.674025; 0.66125 0.714602 0.581433; 0.674025 0.581433 0.784324]),,Char[],Float64[],-Inf,Inf)



We then run the MM algorithm, starting from the Fisher scoring answer. MM finds an improved solution with objective value 8.955397e+03.


```julia
# trait59_model contains the fitted model by Fisher scoring now
@time mle_mm!(trait57_model, trait57_data; verbose=true)
trait57_model
```

    
         MM Algorithm
      Iter      Objective  
    --------  -------------
           0  -1.470099e+04
           1  -1.470099e+04
    
      0.363096 seconds (539.27 k allocations: 20.482 MB, 2.19% gc time)





    VarianceComponentModels.VarianceComponentModel{Float64,2,Array{Float64,2},Array{Float64,2}}(,(
    [0.281163 0.280014 0.232384; 0.280014 0.284899 0.220285; 0.232384 0.220285 0.212687],
    
    [0.716875 0.66125 0.674025; 0.66125 0.714602 0.581433; 0.674025 0.581433 0.784324]),,Char[],Float64[],-Inf,Inf)



Do another run of MM algorithm from default starting point. It leads to a slightly better local optimum -1.470104e+04, slighly worse than the Fisher scoring result. Follow up anlaysis should use the Fisher scoring result.


```julia
# default starting point
trait57_model = VarianceComponentModel(trait57_data)
@time _, _, _, Σcov, = mle_mm!(trait57_model, trait57_data; verbose=true)
trait57_model
```

    
         MM Algorithm
      Iter      Objective  
    --------  -------------
           0  -3.024751e+04
           1  -2.040338e+04
           2  -1.656127e+04
           3  -1.528591e+04
           4  -1.491049e+04
           5  -1.480699e+04
           6  -1.477870e+04
           7  -1.477026e+04
           8  -1.476696e+04
           9  -1.476499e+04
          10  -1.476339e+04
          20  -1.475040e+04
          30  -1.474042e+04
          40  -1.473272e+04
          50  -1.472677e+04
          60  -1.472215e+04
          70  -1.471852e+04
          80  -1.471565e+04
          90  -1.471336e+04
         100  -1.471152e+04
         110  -1.471002e+04
         120  -1.470879e+04
         130  -1.470778e+04
         140  -1.470694e+04
         150  -1.470623e+04
         160  -1.470563e+04
         170  -1.470513e+04
         180  -1.470469e+04
         190  -1.470432e+04
         200  -1.470400e+04
         210  -1.470372e+04
         220  -1.470347e+04
         230  -1.470326e+04
         240  -1.470307e+04
         250  -1.470290e+04
         260  -1.470275e+04
         270  -1.470262e+04
         280  -1.470250e+04
         290  -1.470239e+04
         300  -1.470229e+04
         310  -1.470220e+04
         320  -1.470213e+04
         330  -1.470205e+04
         340  -1.470199e+04
         350  -1.470193e+04
         360  -1.470187e+04
         370  -1.470182e+04
         380  -1.470177e+04
         390  -1.470173e+04
         400  -1.470169e+04
         410  -1.470165e+04
         420  -1.470162e+04
         430  -1.470159e+04
         440  -1.470156e+04
         450  -1.470153e+04
         460  -1.470150e+04
         470  -1.470148e+04
         480  -1.470146e+04
         490  -1.470143e+04
         500  -1.470141e+04
         510  -1.470140e+04
         520  -1.470138e+04
         530  -1.470136e+04
         540  -1.470134e+04
         550  -1.470133e+04
         560  -1.470132e+04
         570  -1.470130e+04
         580  -1.470129e+04
         590  -1.470128e+04
         600  -1.470127e+04
         610  -1.470125e+04
         620  -1.470124e+04
         630  -1.470123e+04
         640  -1.470122e+04
         650  -1.470122e+04
         660  -1.470121e+04
         670  -1.470120e+04
         680  -1.470119e+04
         690  -1.470118e+04
         700  -1.470118e+04
         710  -1.470117e+04
         720  -1.470116e+04
         730  -1.470116e+04
         740  -1.470115e+04
         750  -1.470114e+04
         760  -1.470114e+04
         770  -1.470113e+04
         780  -1.470113e+04
         790  -1.470112e+04
         800  -1.470112e+04
         810  -1.470111e+04
         820  -1.470111e+04
         830  -1.470111e+04
         840  -1.470110e+04
         850  -1.470110e+04
         860  -1.470109e+04
         870  -1.470109e+04
         880  -1.470109e+04
         890  -1.470108e+04
         900  -1.470108e+04
         910  -1.470108e+04
         920  -1.470108e+04
         930  -1.470107e+04
         940  -1.470107e+04
         950  -1.470107e+04
         960  -1.470106e+04
         970  -1.470106e+04
         980  -1.470106e+04
         990  -1.470106e+04
        1000  -1.470106e+04
        1010  -1.470105e+04
        1020  -1.470105e+04
        1030  -1.470105e+04
        1040  -1.470105e+04
        1050  -1.470105e+04
        1060  -1.470104e+04
        1070  -1.470104e+04
        1080  -1.470104e+04
        1090  -1.470104e+04
        1100  -1.470104e+04
        1110  -1.470104e+04
    
      0.846316 seconds (153.65 k allocations: 15.453 MB, 0.89% gc time)





    VarianceComponentModels.VarianceComponentModel{Float64,2,Array{Float64,2},Array{Float64,2}}(,(
    [0.281188 0.280032 0.232439; 0.280032 0.284979 0.220432; 0.232439 0.220432 0.212922],
    
    [0.71685 0.661232 0.67397; 0.661232 0.71452 0.581287; 0.67397 0.581287 0.784092]),,Char[],Float64[],-Inf,Inf)



Heritability from 3-variate estimate and their standard errors.


```julia
h, hse = heritability(trait57_model.Σ, Σcov)
[h'; hse']
```




    2×3 Array{Float64,2}:
     0.281741   0.285122   0.21356  
     0.0778033  0.0773313  0.0841103



## 13-trait joint analysis

In some situations, such as studying the genetic covariance, we need to jointly analyze 13 traits. We first try the **Fisher scoring algorithm**.


```julia
# initialize model parameters
traitall_model = VarianceComponentModel(cg10kdata_rotated)
# estimate variance components using Fisher scoring algorithm
@time mle_fs!(traitall_model, cg10kdata_rotated; solver=:Ipopt, verbose=true)
```

    This is Ipopt version 3.12.4, running with linear solver mumps.
    NOTE: Other linear solvers might be more efficient (see Ipopt documentation).
    
    Number of nonzeros in equality constraint Jacobian...:        0
    Number of nonzeros in inequality constraint Jacobian.:        0
    Number of nonzeros in Lagrangian Hessian.............:    16653
    
    Total number of variables............................:      182
                         variables with only lower bounds:        0
                    variables with lower and upper bounds:        0
                         variables with only upper bounds:        0
    Total number of equality constraints.................:        0
    Total number of inequality constraints...............:        0
            inequality constraints with only lower bounds:        0
       inequality constraints with lower and upper bounds:        0
            inequality constraints with only upper bounds:        0
    
    iter    objective    inf_pr   inf_du lg(mu)  ||d||  lg(rg) alpha_du alpha_pr  ls
       0  1.3113368e+05 0.00e+00 1.00e+02   0.0 0.00e+00    -  0.00e+00 0.00e+00   0 
       5  8.2237394e+04 0.00e+00 6.04e+02 -11.0 2.53e+00    -  1.00e+00 1.00e+00f  1 MaxS
      10  1.2380115e+05 0.00e+00 1.03e+03 -11.0 6.31e+01  -5.4 1.00e+00 1.00e+00h  1 MaxS
      15  1.4133320e+05 0.00e+00 1.99e+02 -11.0 4.54e+02  -7.8 1.00e+00 1.00e+00h  1 MaxS



    Base.LinAlg.PosDefException(25)

    

     in chkposdef at ./linalg/lapack.jl:44 [inlined]

     in sygvd!(::Int64, ::Char, ::Char, ::Array{Float64,2}, ::Array{Float64,2}) at ./linalg/lapack.jl:4908

     in eigfact!(::Symmetric{Float64,Array{Float64,2}}, ::Symmetric{Float64,Array{Float64,2}}) at ./linalg/symmetric.jl:224

     in eigfact(::Symmetric{Float64,Array{Float64,2}}, ::Symmetric{Float64,Array{Float64,2}}) at ./linalg/eigen.jl:274

     in VarianceComponentModels.TwoVarCompModelRotate{T<:AbstractFloat,BT<:Union{AbstractArray{T,1},AbstractArray{T,2}}}(::VarianceComponentModels.VarianceComponentModel{Float64,2,Array{Float64,2},Array{Float64,2}}) at /Users/huazhou/.julia/v0.5/VarianceComponentModels/src/VarianceComponentModels.jl:121

     in eval_f(::VarianceComponentModels.TwoVarCompOptProb{VarianceComponentModels.VarianceComponentModel{Float64,2,Array{Float64,2},Array{Float64,2}},VarianceComponentModels.TwoVarCompVariateRotate{Float64,Array{Float64,2},Array{Float64,2}},Array{Float64,2},Array{Float64,1},VarianceComponentModels.VarianceComponentAuxData{Array{Float64,2},Array{Float64,1}}}, ::Array{Float64,1}) at /Users/huazhou/.julia/v0.5/VarianceComponentModels/src/two_variance_component.jl:683

     in (::Ipopt.#eval_f_cb#4{VarianceComponentModels.TwoVarCompOptProb{VarianceComponentModels.VarianceComponentModel{Float64,2,Array{Float64,2},Array{Float64,2}},VarianceComponentModels.TwoVarCompVariateRotate{Float64,Array{Float64,2},Array{Float64,2}},Array{Float64,2},Array{Float64,1},VarianceComponentModels.VarianceComponentAuxData{Array{Float64,2},Array{Float64,1}}}})(::Array{Float64,1}) at /Users/huazhou/.julia/v0.5/Ipopt/src/IpoptSolverInterface.jl:53

     in eval_f_wrapper(::Int32, ::Ptr{Float64}, ::Int32, ::Ptr{Float64}, ::Ptr{Void}) at /Users/huazhou/.julia/v0.5/Ipopt/src/Ipopt.jl:89

     in solveProblem(::Ipopt.IpoptProblem) at /Users/huazhou/.julia/v0.5/Ipopt/src/Ipopt.jl:304

     in optimize!(::Ipopt.IpoptMathProgModel) at /Users/huazhou/.julia/v0.5/Ipopt/src/IpoptSolverInterface.jl:120

     in #mle_fs!#29(::Int64, ::Symbol, ::Symbol, ::Bool, ::Function, ::VarianceComponentModels.VarianceComponentModel{Float64,2,Array{Float64,2},Array{Float64,2}}, ::VarianceComponentModels.TwoVarCompVariateRotate{Float64,Array{Float64,2},Array{Float64,2}}) at /Users/huazhou/.julia/v0.5/VarianceComponentModels/src/two_variance_component.jl:893

     in (::VarianceComponentModels.#kw##mle_fs!)(::Array{Any,1}, ::VarianceComponentModels.#mle_fs!, ::VarianceComponentModels.VarianceComponentModel{Float64,2,Array{Float64,2},Array{Float64,2}}, ::VarianceComponentModels.TwoVarCompVariateRotate{Float64,Array{Float64,2},Array{Float64,2}}) at ./<missing>:0


From the output we can see the Fisher scoring algorithm ran into some numerical issues. Let's try the **MM algorithm**.


```julia
# reset model parameters
traitall_model = VarianceComponentModel(cg10kdata_rotated)
# estimate variance components using Fisher scoring algorithm
@time mle_mm!(traitall_model, cg10kdata_rotated; verbose=true)
```

    
         MM Algorithm
      Iter      Objective  
    --------  -------------
           0  -1.311337e+05
           1  -8.002195e+04
           2  -5.807051e+04
           3  -4.926234e+04
           4  -4.611182e+04
           5  -4.511727e+04
           6  -4.482798e+04
           7  -4.474410e+04
           8  -4.471610e+04
           9  -4.470285e+04
          10  -4.469355e+04
          20  -4.462331e+04
          30  -4.456960e+04
          40  -4.452834e+04
          50  -4.449652e+04
          60  -4.447178e+04
          70  -4.445237e+04
          80  -4.443699e+04
          90  -4.442467e+04
         100  -4.441470e+04
         110  -4.440656e+04
         120  -4.439985e+04
         130  -4.439427e+04
         140  -4.438959e+04
         150  -4.438564e+04
         160  -4.438229e+04
         170  -4.437941e+04
         180  -4.437694e+04
         190  -4.437480e+04
         200  -4.437294e+04
         210  -4.437131e+04
         220  -4.436989e+04
         230  -4.436863e+04
         240  -4.436751e+04
         250  -4.436652e+04
         260  -4.436564e+04
         270  -4.436485e+04
         280  -4.436414e+04
         290  -4.436351e+04
         300  -4.436293e+04
         310  -4.436242e+04
         320  -4.436195e+04
         330  -4.436152e+04
         340  -4.436113e+04
         350  -4.436078e+04
         360  -4.436046e+04
         370  -4.436016e+04
         380  -4.435989e+04
         390  -4.435965e+04
         400  -4.435942e+04
         410  -4.435921e+04
         420  -4.435902e+04
         430  -4.435884e+04
         440  -4.435867e+04
         450  -4.435852e+04
         460  -4.435838e+04
         470  -4.435825e+04
         480  -4.435813e+04
         490  -4.435802e+04
         500  -4.435791e+04
         510  -4.435781e+04
         520  -4.435772e+04
         530  -4.435764e+04
         540  -4.435756e+04
         550  -4.435748e+04
         560  -4.435741e+04
         570  -4.435735e+04
         580  -4.435729e+04
         590  -4.435723e+04
         600  -4.435718e+04
         610  -4.435713e+04
         620  -4.435708e+04
         630  -4.435704e+04
         640  -4.435700e+04
         650  -4.435696e+04
         660  -4.435692e+04
         670  -4.435688e+04
         680  -4.435685e+04
         690  -4.435682e+04
         700  -4.435679e+04
         710  -4.435676e+04
         720  -4.435674e+04
         730  -4.435671e+04
         740  -4.435669e+04
         750  -4.435667e+04
         760  -4.435665e+04
         770  -4.435663e+04
         780  -4.435661e+04
         790  -4.435659e+04
         800  -4.435657e+04
         810  -4.435656e+04
         820  -4.435654e+04
         830  -4.435653e+04
         840  -4.435651e+04
         850  -4.435650e+04
         860  -4.435649e+04
         870  -4.435648e+04
         880  -4.435647e+04
         890  -4.435646e+04
         900  -4.435645e+04
         910  -4.435644e+04
         920  -4.435643e+04
         930  -4.435642e+04
         940  -4.435641e+04
         950  -4.435640e+04
         960  -4.435639e+04
         970  -4.435639e+04
         980  -4.435638e+04
         990  -4.435637e+04
        1000  -4.435637e+04
        1010  -4.435636e+04
        1020  -4.435635e+04
        1030  -4.435635e+04
        1040  -4.435634e+04
        1050  -4.435634e+04
        1060  -4.435633e+04
        1070  -4.435633e+04
        1080  -4.435632e+04
    
      3.696088 seconds (156.73 k allocations: 69.792 MB, 0.32% gc time)





    (-44356.32043185692,VarianceComponentModels.VarianceComponentModel{Float64,2,Array{Float64,2},Array{Float64,2}}(,(
    [0.273498 0.192141 … -0.128643 -0.098307; 0.192141 0.219573 … -0.0687355 -0.0433724; … ; -0.128643 -0.0687355 … 0.117185 0.0899824; -0.098307 -0.0433724 … 0.0899824 0.106354],
    
    [0.723441 0.568135 … -0.0586259 -0.12469; 0.568135 0.77999 … 0.0236098 0.0464835; … ; -0.0586259 0.0236098 … 0.881707 0.551818; -0.12469 0.0464835 … 0.551818 0.893023]),,Char[],Float64[],-Inf,Inf),(
    [0.0111652 0.0131052 … 0.012894 0.0127596; 0.0131077 0.0151572 … 0.0171615 0.0171431; … ; 0.0128941 0.0171614 … 0.0174141 0.0182039; 0.0127597 0.0171428 … 0.0182038 0.0187958],
    
    [0.011228 0.0133093 … 0.0130109 0.012784; 0.0133113 0.0158091 … 0.0178685 0.0177999; … ; 0.0130107 0.0178687 … 0.017957 0.0187655; 0.0127836 0.0177997 … 0.0187654 0.0193582]),
    [0.000124662 7.24629e-5 … -3.61414e-7 -1.40373e-5; 7.24483e-5 0.000171812 … -2.03586e-5 -3.26457e-6; … ; -3.87301e-7 -2.03778e-5 … 0.000352145 -1.48419e-5; -1.40438e-5 -3.27339e-6 … -1.48381e-5 0.000374741],
    
    ,)



It converges after ~1000 iterations.

## Save analysis results


```julia
#using JLD
#@save "copd.jld"
#whos()
```
