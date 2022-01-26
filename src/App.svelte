<script>
  import {
    Col,
    Container,
    Row,
    Input,
    Alert,
    Button,
    Icon,
    Tooltip,
    Collapse,
  } from "sveltestrap";
  import { fragments, traits } from "./data.js";
  import { copyTextToClipboard } from "./copyToClipboard.js";

  let traitFilterList = [];

  $: filteredFragments =
    traitFilterList.length > 0
      ? fragments.filter((fragment) => {
          let filtered = false;
          for (let i = 0; i < traitFilterList.length; i++) {
            let trait = traitFilterList[i];
            if (softString(fragment.set_effect_1) == softString(trait.name)) {
              filtered = true;
              break;
            }
            if (softString(fragment.set_effect_2) == softString(trait.name)) {
              filtered = true;
              break;
            }
          }
          return filtered;
        })
      : fragments;
  //READ ONLY, FOR VISUAL ONLY, perform all queries on 'fragments'
  $: frags1 = filteredFragments.slice(0, filteredFragments.length / 2);
  $: frags2 = filteredFragments.slice(
    filteredFragments.length / 2,
    filteredFragments.length
  );
  $: fragLists = [frags1, frags2];

  // Change when Jagex patch
  const maxFragments = 7;

  let checked = [];
  let activeTraits = [];
  let invalidQuery = false;
  let limitToMaxFrags = true;

  if ("URLSearchParams" in window) {
    try {
      //parse query string
      let queryList = window.location.search.substring(1).split("=")[1];

      if (queryList && queryList.length > 0) {
        //url decode query
        queryList = decodeURIComponent(queryList);
        //split query into array
        queryList = queryList.split(",");

        // validate
        queryList.forEach((query) => {
          if (isNaN(query)) {
            invalidQuery = true;
            throw new Error("Invalid query");
          }
        });

        queryList.forEach((i) => {
          checked.push(fragments[i]);
        });

        if (checked.length > maxFragments) {
          limitToMaxFrags = false;
        }

        checked = checked;
        updateTraits();
      }
    } catch (e) {
      //Remove in prod
      console.log(e);
    }
  }

  function changeEvent(fragment, e) {
    if (e.target.checked) {
      checked.push(fragment.fragment);
    } else {
      checked.splice(checked.indexOf(fragment.fragment), 1);
    }
    checked = checked;

    updateTraits();
  }

  function relicSetChange(relicSet, e) {
    if (e.target.checked) {
      traitFilterList.push(relicSet.trait);
    } else {
      traitFilterList.splice(traitFilterList.indexOf(relicSet.trait), 1);
    }
    traitFilterList = traitFilterList;
  }

  function updateTraits() {
    let traitCounts = new Map();

    for (let fragmentName in checked) {
      let fragObj = checked[fragmentName];

      let val1 = traitCounts.get(fragObj.set_effect_1);
      let val2 = traitCounts.get(fragObj.set_effect_2);

      if (!val1) {
        traitCounts.set(fragObj.set_effect_1, 1);
      } else {
        traitCounts.set(fragObj.set_effect_1, val1 + 1);
      }
      if (!val2) {
        traitCounts.set(fragObj.set_effect_2, 1);
      } else {
        traitCounts.set(fragObj.set_effect_2, val2 + 1);
      }
    }

    activeTraits = [];

    traitCounts.forEach(function (value, key) {
      traits.forEach((trait) => {
        if (trait.name === key) {
          let traitObj = {
            name: trait.name,
            description: "",
            count: 0,
          };
          trait.effects.forEach((effect) => {
            // if the incoming value from trait counts is higher than the cost and the already set cost if it exists (default 0 garentees it will be set)
            if (value >= effect.cost && value >= traitObj.count) {
              traitObj.description = effect.description;
              traitObj.count = effect.cost;
            }
          });
          if (traitObj.count > 0) {
            activeTraits.push(traitObj);
          }
        }
      });
    });

    activeTraits = activeTraits;

    if ("URLSearchParams" in window) {
      var searchParams = new URLSearchParams(window.location.search);
      let ids = [];
      for (let i = 0; i < checked.length; i++) {
        let checkedFrag = checked[i];
        let index = fragments.indexOf(checkedFrag);
        if (index !== -1) {
          ids.push(index);
        }
      }
      searchParams.set("frag", ids.join(","));
      var newRelativePathQuery =
        window.location.pathname + "?" + searchParams.toString();
      history.pushState(null, "", newRelativePathQuery);
    }
  }

  function isChecked(name) {
    return checked.find((fragment) => fragment.name === name) === undefined
      ? false
      : true;
  }

  function isFilteredSet(name) {
    return traitFilterList.find((relicSet) => relicSet.name === name) ===
      undefined
      ? false
      : true;
  }

  function htmlFriendlyName(name) {
    //Replace all non-alphanumeric characters with an underscore
    return name.replace(/\W/g, "_");
  }

  function softString(st) {
    //Replace spaces with underscores
    let rs = st.replace(/\s/g, "_");
    //Replace all non-alphanumeric characters with an underscore
    rs = rs.replace(/\W/g, "_");
    //lowercase all characters
    rs = rs.toLowerCase();

    return rs;
  }
  function isFiltered(fragment, _traitFilterList) {
    if (traitFilterList.length === 0) {
      //Everything is filtered. yay.
      return true;
    }

    let filtered = false;

    for (let i = 0; i < _traitFilterList.length; i++) {
      let trait = _traitFilterList[i];
      if (softString(fragment.set_effect_1) == softString(trait.name)) {
        filtered = true;
        break;
      }
      if (softString(fragment.set_effect_2) == softString(trait.name)) {
        filtered = true;
        break;
      }
    }

    return filtered;
  }

  function getEffectCostsString(trait) {
    let effectCosts = "(" + trait.effects[0].cost;
    trait.effects.slice(1).forEach((x) => effectCosts += ` | ${x.cost}`);
    return effectCosts + ")";
  }
</script>

<main>
  <Container>
    <Row class="mb-3">
      <Col xs="4" class="text-start">
        <Button color="primary" id="filterSetToggler">Filter Relic Sets</Button>
      </Col>
      <Col xs="4" class="text-center">
        <Input
          type="checkbox"
          label="Limit to {maxFragments}"
          bind:checked={limitToMaxFrags}
          disabled={checked.length > maxFragments && !limitToMaxFrags}
          style="position: absolute"
        />
      </Col>

      <Col xs="4" class="text-end">
        <Button
          color="primary"
          on:click={() => {
            copyTextToClipboard(window.location.href);
          }}
        >
          Copy to Clipboard
        </Button>
      </Col>
    </Row>

    <Collapse toggler="#filterSetToggler" class="mb-3">
      <Row>
        {#each traits as trait}
          <Col xs="4" md="3">
            <Input
              type="checkbox"
              checked={isFilteredSet(trait.name)}
              label="{trait.name} {getEffectCostsString(trait)}"
              on:change={(e) => {
                relicSetChange({ trait }, e);
              }}
            />
          </Col>
        {/each}
      </Row>
    </Collapse>

    {#if invalidQuery}
      <Row>
        <Alert dismissible color="danger">
          <h4 class="alert-heading text-capitalize">
            Query String is invalid! should be format "?frag=1,2,3,4,5,6,7"
          </h4>
        </Alert>
      </Row>
    {/if}
    <Row>
      <Row>
        {#each fragLists as fragmentList}
          <Col xs="6" md="3" class="mb-4">
            {#each fragmentList as fragment}
              {#if isFiltered(fragment, traitFilterList)}
                <Row>
                  <Col xs="10">
                    <Input
                      type="switch"
                      label={fragment.name}
                      on:change={(e) => {
                        changeEvent({ fragment }, e);
                      }}
                      disabled={checked.length >= maxFragments &&
                        !isChecked(fragment.name) &&
                        limitToMaxFrags}
                      checked={isChecked(fragment.name)}
                    />
                  </Col>
                  <Col xs="2">
                    <Icon
                      id={`btn-${htmlFriendlyName(fragment.name)}`}
                      name="info-square-fill d-inline "
                    />
                    <Tooltip
                      target={`btn-${htmlFriendlyName(fragment.name)}`}
                      bottom
                      >{fragment.level_effects} <br /><br /> Set Effect 1: {fragment.set_effect_1}
                      <br />
                      Set Effect 2: {fragment.set_effect_2}</Tooltip
                    >
                  </Col>
                </Row>
              {/if}
            {/each}
          </Col>
        {/each}
        <Col xs="12" md="6">
          <Row>
            <Col xs="6">
              <h2 style="text-decoration:underline;">Selected Fragments</h2>
              {#each checked as frag}
                <Row>
                  <h5>{frag.name}</h5>
                </Row>
              {:else}
                <Row>
                  <h5>No Fragments Selected</h5>
                </Row>
              {/each}
            </Col>
            <Col xs="6">
              <h2 style="text-decoration:underline;">Relic Sets</h2>
              <Row>
                {#each activeTraits as trait}
                  <Row>
                    <h3>{trait.name} ({trait.count})</h3>
                    <p>{trait.description}</p>
                  </Row>
                {/each}
              </Row>
            </Col>
          </Row>
        </Col>
      </Row>
    </Row>
  </Container>
  <Row nogutter class="mt-4">
    <Col
      xs="6"
      md="6"
      class="d-flex justify-content-start text-start align-items-start"
    >
      <a href="https://ko-fi.com/M4M683IMB" rel="noreferrer" target="_blank"
        ><img
          height="36"
          style="border:0px;height:36px;"
          src="https://cdn.ko-fi.com/cdn/kofi5.png?v=3"
          border="0"
          alt="Buy Me a Coffee at ko-fi.com"
        /></a
      >
    </Col>
    <Col xs="6" md="6" class="text-end">
      <a href="https://github.com/JackDallas/RSLeaguesTraits">
        <Icon name="github" style="font-size:30px;color:white;" />
      </a>
    </Col>
  </Row>
</main>
