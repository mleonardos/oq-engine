Rupture sampling: how does it work?
===================================

In this section we explain how the sampling of ruptures in event based
calculations works, at least for the case of Poissonian sources.
As an example, consider the following point source:

>>> from openquake.hazardlib import nrml
>>> src = nrml.get('''\
... <pointSource id="1" name="Point Source"
...              tectonicRegion="Active Shallow Crust">
...     <pointGeometry>
...         <gml:Point><gml:pos>179.5 0</gml:pos></gml:Point>
...         <upperSeismoDepth>0</upperSeismoDepth>
...         <lowerSeismoDepth>10</lowerSeismoDepth>
...     </pointGeometry>
...     <magScaleRel>WC1994</magScaleRel>
...     <ruptAspectRatio>1.5</ruptAspectRatio>
...     <truncGutenbergRichterMFD aValue="3" bValue="1" minMag="5" maxMag="7"/>
...     <nodalPlaneDist>
...         <nodalPlane dip="30" probability="1" strike="45" rake="90" />
...     </nodalPlaneDist>
...     <hypoDepthDist>
...         <hypoDepth depth="4" probability="1"/>
...     </hypoDepthDist>
... </pointSource>''', investigation_time=1, width_of_mfd_bin=1.0)

The source here is particularly simple, with only one
seismogenic depth and one nodal plane. It generates two ruptures,
because with a ``width_of_mfd_bin`` of 1 there are only two magnitudes in
the range from 5 to 7:

>>> [(mag1, rate1), (mag2, rate2)] = src.get_annual_occurrence_rates()
>>> mag1
5.5
>>> mag2
6.5

The occurrence rates are respectively 0.009 and 0.0009. So, if we set
the number of stochastic event sets to 1,000,000

>>> num_ses = 1_000_000

we would expect the first rupture (the one with magnitude 5.5) to
occur around 9,000 times and the second rupture (the one with magnitude
6.5) to occur around 900 times. Clearly the exact numbers will depend on
the stochastic seed; if we set

>>> import numpy.random
>>> numpy.random.seed(42)

then we will have (for ``investigation_time = 1``)

>>> numpy.random.poisson(rate1 * num_ses * 1)
8966
>>> numpy.random.poisson(rate2 * num_ses * 1)
921

These are the number of occurrences of each rupture in the effective
investigation time, i.e. the investigation time multiplied by the
number of stochastic event sets and the number of realizations (here we
assumed 1 realization).

The total number of events generated by the source will be

``number_of_events = sum(n_occ for each rupture)``

i.e. 8,966 + 921 = 9,887, with ~91% of the events associated to the first
rupture and ~9% of the events associated to the second rupture.

Since the details of the seed algorithm can change with updates to the
the engine, if you run an event based calculation with the same
parameters with different versions of the engine, you may not get 
exactly the same number of events, but something close given a reasonably
long effective investigation time. After running the calculation, inside
the datastore, in the ``ruptures`` dataset you will find the two
ruptures, their occurrence rates and their integer number of
occurrences (``n_occ``). If the effective investigation time is large
enough the relation

``n_occ ~ occurrence_rate * eff_investigation_time``

will hold. If the effective investigation time is not large enough, or the
occurrence rate is extremely small, then you should expect to see larger
differences between the expected number of occurrences and ``n_occ``, 
as well as a strong seed dependency.

It is important to notice than in order to determine the effective
investigation time, the engine takes into account also the ground motion
logic tree and the correct formula to use is

``eff_investigation_time = investigation_time * num_ses * num_rlzs``

where ``num_rlzs`` is the number of realizations in the 
ground motion logic tree.

Just to be concrete, if you run a calculation with the same parameters
as described before, but with two GMPEs instead of one (and
``number_of_logic_tree_samples = 0``), then the total number of paths
admitted by the logic tree will be 2 and you should expect to get
about twice the number of occurrences for each rupture.
Users wanting to know the nitty-gritty details should look at the
code, inside hazardlib/source/base.py, in the method
``src.sample_ruptures(eff_num_ses, ses_seed)``.

The case of multiple tectonic region types and realizations
-----------------------------------------------------------

Since engine 3.13 hazardlib contains some helper functions that
allow users to compute stochastic event sets manually. Such functions
are in the module `openquake.hazardlib.calc.stochastic`. Internally,
the engine does not use directly such functions, since it needs to
follow a slightly more complex logic in order to make the calculations
parallelizable. Also, the engine is able to manage general source model
logic trees, while the helper functions are meant to work in a situation
with a single source model and a trivial source model logic tree.
However, in spirit, the idea is the same.

As a concrete example, consider the event based logic tree demo
which is part of the engine distribution (search for
demos/hazard/EventBasedPSHA). This is a case with a trivial
source model logic tree, a source model with two tectonic region
types and a GSIM logic tree generating 2x2 = 4 realizations with
weights .36, .24, .24, .16 respectively. The effective investigation
time is

``eff_time = 50 years x 250 ses x 4 rlz = 50,000 years``

You can sample the ruptures with the following commands,
assuming you are inside the demo directory::

 >> from openquake.hazardlib.contexts import ContextMaker
 >> from openquake.commonlib import readinput
 >> from openquake.hazardlib.calc.stochastic import sample_ebruptures
 >> oq = readinput.get_oqparam('job.ini')
 >> gsim_lt = readinput.get_gsim_lt(oq)
 >> csm = readinput.get_composite_source_model(oq)
 >> rlzs_by_gsim_trt = gsim_lt.get_rlzs_by_gsim_trt(
 ..     oq.number_of_logic_tree_samples, oq.random_seed)
 >> cmakerdict = {trt: ContextMaker(trt, rbg, vars(oq))
 ..                    for trt, rbg in rlzs_by_gsim_trt.items()}
 >> ebruptures = sample_ebruptures(csm.src_groups, cmakerdict)

Then you can extract the events associated to the ruptures with
the function `get_ebr_df` which returns a DataFrame::

  >> from openquake.hazardlib.calc.stochastic import get_ebr_df
  >> ebr_df = get_ebr_df(ebruptures, cmakerdict)

This DataFrame has fields `eid` (event ID) and `rlz` (realization number)
and it is indexed by the ordinal of the rupture. For instance it can be
used to determine the number of events per realization::

 >> ebr_df.groupby('rlz').count()
 eid   rlz      
 0    7842
 1    7709
 2    7893
 3    7856

Notice that the number of events is more or less the same for each realization.
This is a general fact, valid also in the case of sampling, a consequence
of the random algorithm used to associate the events to the realizations.

The difference between full enumeration and sampling
--------------------------------------------------------------

Users are often confused about the difference between full enumeration and
sampling. For this reason the engine distribution comes
with a pedagogical example that considers an extremely simplified situation
comprising a single site, a single rupture, and only two GMPEs.
You can find the example in the engine repository under the directory
`openquake/qa_tests_data/event_based/case_3`. If you look at the ground motion
logic tree file, the two GMPEs are AkkarBommer2010 (with weight 0.9)
and SadighEtAl1997 (with weight 0.1).

The parameters in the job.ini are::

 investigation_time = 1
 ses_per_logic_tree_path = 5_000
 number_of_logic_tree_paths = 0

Since there are 2 realizations, the effective investigation time is
10,000 years. If you run the calculation, you will generate (at least
with version 3.13 of the engine, though the details may change with the version)
10,121 events, since the occurrence rate of the rupture was chosen to be 1.
Roughly half of the events will be associated with the first GMPE
(AkkarBommer2010) and half with the second GMPE (SadighEtAl1997).
Actually, if you look at the test, the precise numbers will be
5,191 and 4,930 events, i.e. 51% and 49% rather than 50% and 50%, but this
is expected and by increasing the investigation time you can get closer
to the ideal equipartition. Therefore, even if the AkkarBommer2010 GMPE
is assigned a relative weight that is 9 times greater than SadighEtAl1997, 
*this is not reflected in the simulated event set*. 
It means that when performing a computation (for instance
to compute the mean ground motion field, or the average loss) one
has to keep the two realizations distinct, and only at the end to
perform the weighted average.

The situation is the opposite when sampling is used. In order to get the
same effective investigation time of 10,000 years you should change the
parameters in the job.ini to::

 investigation_time = 1
 ses_per_logic_tree_path = 1
 number_of_logic_tree_paths = 10_000

Now there are 10,000 realizations, not 2, and they *all have the same
weight .0001*. The number of events per realization is still roughly
constant (around 1) and there are still 10,121 events, however now *the
original weights are reflected in the event set*.  In particular there
are 9,130 events associated to the AkkarBommer2010 GMPE and 991 events
associated to the SadighEtAl1997 GMPE. There is no need to keep the realizations
separated: since they have all the same weigths, you can trivially
compute average quantities. AkkarBommer2010 will count more than SadighEtAl1997
simply because there are 9 times more events for it (actually 9130/991 = 9.2,
but the rate will tend to 9 when the effective time will tend to infinity).

NB: just to be clear, normally realizations are not in one-to-one
correspondence with GMPEs. In this example, it is true because there is
a single tectonic region type. However, usually there are multiple tectonic
region types, and a realization is associated to a tuple of GMPEs.
