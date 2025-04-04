---
layout: post
title: Creating herbarium labels with typst
---

For a recent description of a novel lichen species I had to create herbarium labels for the specimens which I had collected.
I have always wanted to automate this so that I would not have to create all the labels manually in LibreOffice.

I have experimented with the type setting system [typst](https://github.com/typst/) for a [preprint](https://www.biorxiv.org/content/10.1101/2023.09.22.558970v1) and I thought that this could be a great opportunity to apply this to my problem.

# Current state of my herbarium database

My herbarium database is a simple spreadsheet. Newly entered specimens receive a unique number and additional information such as, genus and species, coordinates, habitat descriptions, etc. 

This is a section of my current database:
![HerbariumDB](https://github.com/reslp/blog/raw/master/images/herbariumdb.png)

When I wanted to create herbarium labels in the past, I just transferred the content of the cells into a word processor and arranged everyting around until it looked (reasonably) ok:
![Label example](https://github.com/reslp/blog/raw/master/images/herbariumlabelexample.png)

However I could also export the database as CSV and then parse it using some script to generate labels automatically. This is how it looks once exported:

![HerbariumDBCSV](https://github.com/reslp/blog/raw/master/images/herbariumdbcsv.png)


# Typst to the rescue

Fortunately typst offers a great scripting capabilities including a way to load CSV files and programatically analyze and typset its content. I was not quite sure how to start though. So I searched for similar examples online and took inspiration from a [typst script](https://typst.app/project/r887qshXMcIWXjhBypuELQ) on how to create conference name tags automatically. My data was a little bit more complex then this example but it was a nice basis for what I wanted to do.

Here is the typst script I came up with:

```
// order of columns is supposed to be: 1:herb number, 2:year, 3:month, 4:day, 5:country, 6:area, 7:collection spot, 8:GPS-Waypoint, 9:lat, 10:long, 11:altitude, 12:description, 13:genus, 14:species, 15:substrate, 16:collectors, 17:n capsules, 18:notes
#let label(number, year, month, day, country, area, spot, waypoint, lat, long, altitude, description,genus, species,substrate, collectors, ncapsules, notes) = {
  let main-color = luma(255) //white
  let secondary-color = luma(0) //black

  let header = [
    #align(center)[#v(3mm)#text(size: 12pt, tracking: 0.6pt, weight: "bold")[Herbarium Philipp Resl Nr.#number]]
  ]

  let spname = align(center, box(inset: 1mm)[
    #set align(center)
    #set par(leading: 0.25em)
    #text(size:17pt, style:"italic", [#genus #species])
  ])
  let desc = align(left, box(inset: 1mm)[
   #set par(leading: 0.25em)
    #set align(left)
    #text(size:12pt, [*#country*: #area, #spot, #lat #long, #altitude m a.s.l., #description. \ Substrate: #substrate])
    //#v(-5mm)
  ])

  let notess = align(left, box(inset: 1mm)[
    #set par(leading: 0.25em)
    #text(size:12pt, [*Notes*: #notes])
  ])

  let colldate = align(left, box(inset: 1mm)[
    #set align(left)
    #text(size:12pt, [#h(2mm) #day.#month.#year #h(1fr) leg. #collectors #h(2mm)])
  ])

    // A single herbarium label consists of the different parts(header, spname, desc, notess, colldate) created above with different formating, texsizes, etc.. These parts are then combined here.
  let content = [
    // arrange content
    #header
    #v(1fr)
    #spname
    #v(1fr)
    #desc
    #notess
    #v(1fr)
    #colldate
    #v(1fr)
  ]

  box(
    height: 80mm,
    width: 130mm,
    clip: true,
    inset: 3mm,
    stroke: .1mm,
    content
  )
}


// set page dimensions and landscape format
#set page(paper: "a4", margin: 10mm, flipped: true)

// load database file
#let specimens = csv("database.csv", delimiter: ";")

// only actually print labels for the last N entries
// this is to avoid creating a PDF from the whole database all the time.
#let N = 6
#let totallen = specimens.len()
#let subsamples = specimens.slice(totallen - N, totallen)

// create variable to store the individual labels
#let labels = []
#for p in subsamples {
  // order of columns is supposed to be: 1:herb number, 2:year, 3:month, 4:day, 5:country, 6:area, 7:collection spot, 8:GPS-Waypoint, 9:lat, 10:long, 11:altitude, 12:description, 13:genus, 14:species, 15:substrate, 16:collectors, 17:n capsules, 18:notes
  lebels = labels + label(p.at(0), p.at(1), p.at(2), p.at(3), p.at(4), p.at(5), p.at(6) , p.at(7), p.at(8), p.at(9), p.at(10), p.at(11), p.at(12), p.at(13), p.at(14), p.at(15), p.at(16), p.at(17)) + box(height: 80mm, width: 1mm,clip: true)
}

#set par(leading: 1mm, spacing: 1mm)
#align(center, labels)
```

# Creating typst labels

I use typst in a docker container. I compile the above script (which is called `label.typ`) using the following command:

```
docker run -v $(pwd):/data --rm --user $(id -u):$(id -g) -it reslp/typst:0.13.1 typst compile label.typ
```


This creates a PDF with the labels:


![TypstLabel](https://github.com/reslp/blog/raw/master/images/herbariumlabeltypst.png)


Much better I think!

I still want to find a better way to select which labels I would like to create from the database but since I usually only create labels from specimens entered last, the current approach works.



 

