compare
================
Gina

# Introduction to irrigation energy

There may be two sources of water:

1.  surface

2.  ground

Ground water will have a ‘head’ associated with the depth of the well.
Both will have ‘head’ associated with the pump pressure.  

The main components defining the irrigation energy use are:

1.  The pump pressure and, if ground water, the well depth

2.  The energy source for the pumping/moving of water (diesel,
    electricity, solar, etc.)

3.  The amount of irrigation water applied

# Scenario

- A flood irrigated field

- Diesel powered pump

- 200 foot well

- 25 PSI pump

- \$1 energy unit (to make back-calculating the amount of fuel easy)

- Alfalfa

- 1 acre

- 50 ac-in applied per acre

- The zipcode is used to auto-fill some NRCS values which we overwrite.
  It doesn’t impact the calculations if you overwrite them. However I
  put in 93603.

  ``` r
  well_depth_ft <- 200
  pump_press_psi <- 25
  water_applied_in_ac <- 50
  ```

# NRCS tool

The NRCS has a tool that will estimate the energy used for irrigation:
<https://ipat.sc.egov.usda.gov/Default.aspx>

When using the above inputs, the estimated energy cost is \$67. I take
this to mean I will have used 67 gallons of diesel:

``` r
nrcs_gal_dies_ac <- 67

ac_per_ha <- 1/0.404

(nrcs_gal_dies_ha <- nrcs_gal_dies_ac * ac_per_ha)
```

    [1] 165.8416

According to Table 1 of FTM, 1 gallon of diesel contains 138,490 BTUs.
We can calculate the number of BTUs we used per hectare:

``` r
btu_per_gal_dies <- 138490

nrcs_btu_per_ha <- nrcs_gal_dies_ha * btu_per_gal_dies

prettyNum(nrcs_btu_per_ha, big.mark = ",", scientific = F)
```

    [1] "22,967,401"

In the tens of millions of BTUs per hectare.

Just for ballpark estimates, Table 1 of FTM says there are 22.7 lbs of
co2e per gallon of diesel (although this is from combustion of the fuel,
only).

``` r
lbsco2_per_gal_dies <- 22.7
kg_per_lb <- 1/2.2

nrcs_kgco2_ha <- nrcs_gal_dies_ac * ac_per_ha * lbsco2_per_gal_dies * kg_per_lb

prettyNum(nrcs_kgco2_ha, big.mark = ",", scientific = F)
```

    [1] "1,711.184"

Irrigation releases co2 on the order of Mg.

# FTM equations

The total head is expressed as the sum of the head from the pump
pressure and the pumping depth (Latex sucks in this environment):

head \[m\] = pumping pressure \[m\] + well depth \[m\]

To calculate the head we therefore just need to do some unit
conversions.

``` r
m_per_psi <- 0.703
m_per_ft <- 0.305

pump_press_m <- pump_press_psi * m_per_psi
well_depth_m <- well_depth_ft * m_per_ft

head_m <- pump_press_m + well_depth_m
print(round(head_m, 0))
```

    [1] 79

This is then used in the pumping energy equation:

$$energy = \frac{H*C_{pump}*(acre-inches_{applied})*25.4 \frac{mm}{in} * (number-of-acres-in-field) * (0.404 \frac{ha}{acre})}{E_{pump}*E_{irrigation}*E_{gearhead}*E_{power-unit}}$$

Where:

$$H = head\\
C_{pump} = 0.0979\\
E_{pump} = 0.75\\
E_{irrigation} = 1\\
E_{gearhead} = 0.95\\
E_{power-unit} = 1$$

The units are very strange, and I cannot follow what is being canceled.
Apparently this produces a value in MJs per hectare? The mm per inch
conversion is very strange.

Constants provided by FTM (no citations):

``` r
# constants, no idea of units
c_pump <- 0.0979
e_pump <- 0.75
e_irr <- 1
e_gear <- 0.95
e_power <- 1

# unit conversions
mm_per_in <- 25.4
m_per_in <- mm_per_in/100
ha_per_ac <- 0.404
btu_per_mj <- 947.8
```

Let’s work through the numerator:

``` r
(num1 <- head_m * c_pump * water_applied_in_ac * mm_per_in * ha_per_ac)
```

    [1] 3946.864

The denominator:

``` r
(den1 <- e_pump * e_irr * e_gear * e_power)
```

    [1] 0.7125

This gives MJ/ha

``` r
(ftm_mj_per_ha <- num1/den1)
```

    [1] 5539.458

Convert to BTUs per ha

``` r
ftm_btu_per_ha <- ftm_mj_per_ha * btu_per_mj
prettyNum(ftm_btu_per_ha, big.mark = ",", scientific = F)
```

    [1] "5,250,299"

So it estimates about half the amount the NRCS does?

What if we assume perfect efficiencies in the denominator?

``` r
ftm_btu_per_ha2 <- num1 * btu_per_mj
prettyNum(ftm_btu_per_ha2, big.mark = ",", scientific = F)
```

    [1] "3,740,838"

What if we convert the amount of water into m instead of in?

``` r
num2 <- head_m * c_pump * water_applied_in_ac * m_per_in * ha_per_ac
ftm_btu_per_ha3 <- num2 * btu_per_mj
prettyNum(ftm_btu_per_ha3, big.mark = ",", scientific = F)
```

    [1] "37,408.38"

Put them in a table to compare. I don’t understand why the NRCS values
are so much higher than the FTM values.

``` r
res <- tibble(calc = c("nrcs", 
                "ftm mm conv", 
                "ftm mm conv 100% eff", 
                "ftm m conv 100% eff"),
       btu_per_ha = c(nrcs_btu_per_ha,
                      ftm_btu_per_ha,
                      ftm_btu_per_ha2,
                      ftm_btu_per_ha3)) %>% 
  mutate(gal_dies_ha = btu_per_ha * 1/btu_per_gal_dies,
         gal_dies_ac = gal_dies_ha * 1/ac_per_ha) %>% 
  mutate_if(is.numeric, round, 0)
res
```

    # A tibble: 4 × 4
      calc                 btu_per_ha gal_dies_ha gal_dies_ac
      <chr>                     <dbl>       <dbl>       <dbl>
    1 nrcs                   22967401         166          67
    2 ftm mm conv             5250299          38          15
    3 ftm mm conv 100% eff    3740838          27          11
    4 ftm m conv 100% eff       37408           0           0
