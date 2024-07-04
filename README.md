 //EditGeofence
import { useForm } from "react-hook-form";
import Button from "../../../Components/Button/Button";
import DropDownMenu from "../../../Components/DropDownMenu/DropDownMenu";
import InputForm from "../../../Components/InputForm/InputForm";
import {
  DrawingManagerF,
  GoogleMap,
  LoadScript,
  PolygonF,
} from "@react-google-maps/api";
import { useCallback, useEffect, useRef, useState } from "react";
import { useGetSitesQuery } from "../../../redux/query/getSitesQuery";
import {
  useAddGeofenceAreaMutation,
  useGetGeofenceDetailsQuery,
  useUpdateGeofenceMutation,
} from "../../../redux/query/geofenceQuery";
import Modal from "../../../Components/Modal/Modal";
import { useGetCustomeFormsQuery } from "../../../redux/query/getCustomeForms";
import { useNavigate, useSearchParams } from "react-router-dom";
import { toast } from "react-toastify";
import { IoMdClose } from "react-icons/io";
import { FaChevronDown } from "react-icons/fa";
const libraries = ["drawing"];
// --polygon options--

const options = {
  drawingControl: true,
  drawingControlOptions: {
    drawingMode: ["polygon"],
  },
  polygonOptions: {
    strokeColor: `#fb6d48d1`,
    fillColor: `#fb6d4861`,
    fillOpacity: 1,
    strokeWeight: 5,
    clickable: true,
    editable: true,
    draggable: true,
    zIndex: 1,
  },
};

const createHourData = () => {
  const hours = [];
  for (let i = 0; i <= 12; i++) {
    hours.push({
      id: i,
      name: `${i} Hour${i > 1 ? "s" : ""}`, // Correct pluralization
      value: i,
    });
  }
  return hours;
};

const hours = createHourData();

const createMinuteData = () => {
  const minutes = [];
  for (let i = 0; i < 60; i++) {
    minutes.push({
      id: i.toString(),
      name: `${i} Min`,
      value: i,
    });
  }
  return minutes;
};

const min = createMinuteData();

const EditGeofence = () => {
  const mapRef = useRef(null);

  const navigate = useNavigate();
  const [searchParams] = useSearchParams();
  const { data: siteList } = useGetSitesQuery({ page: "", search: "" });
  const [selectOptions, setSelectOptions] = useState();
  console.log(selectOptions, "selectOptions");
  const [modalStatus, setModalStatus] = useState(false);
  const {
    register,
    handleSubmit,
    setValue,
    reset,
    formState: { errors },
  } = useForm();

  const {
    register: register2,
    formState: { errors: errors2 },
    handleSubmit: handleSubmit2,
  } = useForm({
    mode: "onBlur",
  });

  const [polyLineData, setPolyLineData] = useState([]);
  const [showFormOptions, setShowFormOptions] = useState(false);
  const [selectFormId, setSelectFormId] = useState();
  const [subGeofaceModel, setSubGeofaceModel] = useState(false);
  const id = searchParams.get("id");
  const [formPramission, setFormPramission] = useState();
  const [defuLetLog, setDefuLetLog] = useState();

  const { data: geofaceDetailsData, isLoading } =
    useGetGeofenceDetailsQuery(id);
  const [colorValue, setColorValue] = useState();
  const [, setGeofaceDetails] = useState({
    geofence_name: "",
    allowable_time: "",
  });

  // ---  api ---
  const [addGeofenceArea] = useAddGeofenceAreaMutation(); // Update CustomeForm

  const [updateGeofence] = useUpdateGeofenceMutation();
  const { data } = useGetCustomeFormsQuery({ page: "", search: "" }); // get CustomeForm Details List

  const updateGeofenceApi = async (data) => {
    const hours = data.allowable_hour;
    const min = data.allowable_min;

    const times = `${hours}:${min}`;
    const filterSite =
      siteList.data.data &&
      siteList?.data?.data?.filter((e) => e.id === selectOptions?.value);
    setFormPramission(filterSite[0]);

    if (hours === min) {
      toast.error("Hour and Min must be different");
    } else {
      try {
        const res = await updateGeofence({
          id: id,
          site_id: selectOptions.value,
          geofence_name: data.geofence_name,
          allowable_time: times,
        }).unwrap();

        toast.success("Update Successfully");
        if (res.success == 200) {
          setSubGeofaceModel(false);
        }
      } catch (error) {
        console.error("error", error);
      }
    }
  };

  const subGeofanceData = async (data) => {
    try {
      let requestData = {
        geofence_id: id,
        geofence_name: data.geofence_form_name,
        color: colorValue,
        form_type: data.form_type,
        Geofencearea: polyLineData,
      };

      if (data.form_type !== "queing") {
        requestData.custome_form_id = selectFormId?.id;
      }

      const res = await addGeofenceArea(requestData).unwrap();
      console.log(res, "res");
      if (res.success == 200) {
        setSubGeofaceModel(false);
        window.location.reload();
      }
    } catch (error) {
      console.error("error", error);
    }
  };

  const handleFormTypeChange = (event) => {
    const selectedFormType = event.target.value;
    if (selectedFormType === "custome") {
      setShowFormOptions(true);
    } else {
      setShowFormOptions(false);
    }
  };
  // --get polygon data and set--
  const setPolygonData = useCallback(async (e) => {
    setSubGeofaceModel(true);
    const area = [];
    console.log(area, "area");
    const polygonData = e.getPath().getArray();
    await polygonData.forEach((p) => {
      const latitude = p.lat();
      const longitude = p.lng();
      area.push({ latitude, longitude });
    });
    setPolyLineData(area);
  }, []);

  useEffect(() => {
    if (geofaceDetailsData) {
      const mainGeoface = geofaceDetailsData?.data?.data?.map(
        (e) => e?.geofence_area[0]?.subgeofence[0]
      );

      const getLetLog = mainGeoface?.map((list) => {
        return {
          lat: list?.latitude,
          lng: list?.longitude,
        };
      });
      //  --

      const averageLng =
        (Math.max(getLetLog.map((e) => e.lng)) +
          Math.min(getLetLog.map((e) => e.lng))) /
        2;
      const averageLat =
        (Math.max(getLetLog.map((e) => e.lat)) +
          Math.min(getLetLog.map((e) => e.lat))) /
        2;

      setDefuLetLog({ lat: averageLat, lng: averageLng });
      // --
      // setDefuLetLog(getLetLog[0]);
    }
  }, [geofaceDetailsData]);

  useEffect(() => {
    if (geofaceDetailsData?.data?.data) {
      const times = geofaceDetailsData?.data?.data[0]?.allowable_time;
      const hour = times.split(":")[0];
      const min = times.split(":")[1];

      setGeofaceDetails({
        geofence_name: geofaceDetailsData?.data?.data[0]?.geofence_name || "",
        allowable_time: geofaceDetailsData?.data?.data[0]?.allowable_time || "",
      });
      setValue(
        "geofence_name",
        geofaceDetailsData.data.data[0].geofence_name || ""
      );
      setValue(
        "allowable_time",
        geofaceDetailsData.data.data[0].allowable_time || ""
      );

      setValue("allowable_hour", Number(hour));
      setValue("allowable_min", Number(min));

      setSelectOptions({
        name: geofaceDetailsData.data.data[0].site_name,
        value: geofaceDetailsData.data.data[0].site_id,
      });
    }
  }, [geofaceDetailsData, setValue]);

  const onLoad = useCallback(
    function callback(map) {
      const locations =
        geofaceDetailsData?.data.data[0].geofence_area[0]?.subgeofence?.map(
          (e) => {
            return { lat: Number(e.latitude), lng: Number(e.longitude) };
          }
        );

      if (locations) {
        //Type definitions for non-npm package Google Maps JavaScript API 3.50
        var bounds = new window.google.maps.LatLngBounds();
        locations.forEach((location) => {
          bounds.extend({
            lat: Number(location.lat),
            lng: Number(location.lng),
          });
        });
        map.fitBounds(bounds); // <-------- resize based on markers, native
      }
    },
    [geofaceDetailsData?.data.data]
  );

  if (isLoading) {
    return <div>Loading...</div>;
  }

  return (
    <>
      {/* ---- main content ---- */}
      <div className="Add_Geofences_Popup_main">
        <div className="Add_Geofences_Popup_upper">
          <div className="Add_Geofences_Popup_left_head">
            <p>Update Geofence</p>
          </div>

          <form onSubmit={handleSubmit(updateGeofenceApi)}>
            <div className="Add_Geofences_Popup_left_form">
              <DropDownMenu
                site="site_name"
                defaultValue="Select Site"
                label="Site"
                listValue={siteList?.data?.data ? siteList?.data?.data : []}
                selectOptions={selectOptions}
                setSelectOptions={setSelectOptions}
              />
              <div
                style={{
                  display: "flex",
                  flexDirection: "column",
                  width: "100%",
                }}
              >
                <InputForm
                  label="Enter Geofence Name"
                  placeholders="Enter Geofence Name"
                  type="text"
                  register={{
                    ...register("geofence_name", {
                      required: "Geofence Name Is Required",
                    }),
                  }}
                />
                {errors.geofence_name && (
                  <p style={{ color: "red" }}>{errors.geofence_name.message}</p>
                )}
              </div>
              <div
                style={{
                  display: "flex",
                  flexDirection: "column",
                  width: "100%",
                }}
              >
                <h3>Enter Allowable Time (In hrs)</h3>

                <div
                  className="datePicker"
                  style={{
                    display: "flex",
                    alignItems: "center",
                    justifyContent: "center",
                    gap: "10px",
                    marginTop: "12px",
                  }}
                >
                  <div style={{ width: "50%" }}>
                    <select
                      name=""
                      id=""
                      {...register("allowable_hour")}
                      style={{
                        width: "100%",
                        height: "60px",
                        fontSize: "16px",
                        paddingLeft: "15px",
                        borderRadius: "5px",
                      }}
                    >
                      {hours.map((e, i) => (
                        <option key={i} value={e.value}>
                          {e.name}
                        </option>
                      ))}
                    </select>
                  </div>

                  <div style={{ width: "50%" }}>
                    <select
                      name=""
                      id=" "
                      {...register("allowable_min")}
                      style={{
                        width: "100%",
                        height: "60px",
                        fontSize: "16px",
                        paddingLeft: "15px",
                        borderRadius: "5px",
                      }}
                    >
                      {min.map((E) => (
                        <option key={E.value} value={E.value}>
                          {E.name}
                        </option>
                      ))}
                    </select>
                  </div>
                </div>
              </div>
            </div>
            <div className="Add_Geofences_Popup_left_form_footer_button">
              <Button
                className="cancel"
                name={"Cancel"}
                type={"button"}
                fun={() => navigate(-1)}
              />
              <Button name={"Update"} />
            </div>
          </form>
        </div>

        <div className="Add_Geofences_Popup_lower">
          <div className="Add_Geofences_Popup_right_head">
            <p>Select Geofence Area</p>
          </div>
          <div className="Add_Geofences_Popup_right_middle">
            <div className="Add_Geofences_Popup_right_map_div">
              <LoadScript
                googleMapsApiKey="AIzaSyC_eaiab1Jet34YC6mXWdfUylWXw1LUnlc"
                libraries={libraries}
              >
                <GoogleMap
                  mapContainerStyle={{ width: "100%", height: "100%" }}
                  ref={mapRef}
                  center={
                    defuLetLog
                      ? defuLetLog
                      : { lat: -35.473469, lng: 149.012375 }
                  }
                  zoom={12}
                  onClick={(e) => {
                    if (e.latLng.lat() && e.latLng.lng()) {
                      setPolyLineData({
                        lng: e.latLng.lng(),
                        lat: e.latLng.lat(),
                      });
                    }
                  }}
                  onLoad={onLoad}
                >
                  <DrawingManagerF
                    drawingMode={"polygon"}
                    options={options}
                    draggable={true}
                    editable={true}
                    onPolygonComplete={setPolygonData}
                  />

                  {isLoading ? (
                    <h1>Loading...</h1>
                  ) : (
                    <>
                      {geofaceDetailsData?.data.data &&
                        geofaceDetailsData?.data.data[0].geofence_area.map(
                          (list, i) => (
                            <PolygonF
                              key={i}
                              path={list?.subgeofence.map((coord) => ({
                                lat: coord.latitude,
                                lng: coord.longitude,
                              }))}
                              options={{
                                fillColor: list?.color || "yellow",
                                strokeColor: list?.color || "#d35400",
                              }}
                            />
                          )
                        )}
                    </>
                  )}
                </GoogleMap>
              </LoadScript>
            </div>

            <div
              className="Add_Geofences_Popup_left_form_footer"
              style={{
                height: "400px",
                display: "flex",
                flexDirection: "column",
                justifyContent: "space-between",
              }}
            >
              {geofaceDetailsData?.data.data[0].geofence_area &&
              geofaceDetailsData?.data.data[0].geofence_area.length > 0 ? (
                <>
                  {geofaceDetailsData?.data.data[0].geofence_area.map(
                    (li, i) => (
                      <div
                        key={i}
                        style={{
                          display: "flex",
                          justifyContent: "space-between",
                          alignItems: "center",
                          paddingBottom: "6px",
                          borderBottom: "1px solid black",
                          marginTop: "10px",
                        }}
                      >
                        <div>
                          <h3>{li.geofence_name}</h3>
                          <h6 style={{ marginTop: "10px", fontSize: "14PX" }}>
                            (
                            {li?.form_type === "custome" &&
                              li?.custome_form_name}
                            {li.form_type === "queing" && "queing"}
                            {li.form_type === "noform" && "noform"})
                          </h6>
                        </div>

                        <div>
                          <div
                            style={{
                              width: "15px",
                              height: "15px",
                              background: li?.color,
                              borderRadius: "50%",
                            }}
                          />
                        </div>
                      </div>
                    )
                  )}
                </>
              ) : (
                <>o Data Found</>
              )}

              <div
                className="Add_Geofences_Popup_left_form_footer_button"
                style={{
                  display: "flex",
                  gap: "15px",
                  justifyContent: "flex-end",
                  marginTop: "20px",
                }}
              >
                <Button name={"Go Back"} fun={() => navigate(-1)} />
              </div>
            </div>
          </div>
        </div>
      </div>

      {/* --- popup content --- */}
      {/* CSS Writen In [_addcompanypopup.scss] File */}
      <Modal status={subGeofaceModel}>
        <div className="Add_Company">
          <div className="Add_Company_Popup_head">
            <p>Add Geofence Details</p>
          </div>

          <form onSubmit={handleSubmit2(subGeofanceData)}>
            <div className="Add_Company_Popup_form">
              <InputForm
                label="Geofence Form Name"
                placeholders="Enter Geofence Form Name"
                type="text"
                register={{ ...register2("geofence_form_name") }}
                onChange={handleFormTypeChange}
              />
            </div>

            <div className="Add_Company_Popup_form_checkbox_head_div">
              <p>Select Form</p>
            </div>

            <div className="Add_Company_Popup_form_checkbox_formSelectOption_div">
              <div className="Add_Company_Popup_form_checkbox_div">
                <InputForm
                  style={{
                    opacity: formPramission?.queuing_function === 0 ? 0.5 : 1,
                  }}
                  disabled={
                    formPramission?.queuing_function === 0 ? true : false
                  }
                  label="Add Custom Form"
                  type="radio"
                  value={"custome"}
                  name="form_type" // Adding the same name for grouping
                  register={{ ...register2("form_type") }}
                  onChange={handleFormTypeChange}
                />

                <div className="Add_Geofences_Popup_left_form_footer_select_form">
                  {showFormOptions && (
                    <>
                      <li
                        style={{
                          cursor: "pointer",
                          display: "flex",
                          alignItems: "center",
                          gap: "10px",
                        }}
                      >
                        <span onClick={() => setModalStatus(true)}>
                          {selectFormId
                            ? selectFormId.name
                            : "Select Custom Form"}
                        </span>
                        {selectFormId ? (
                          <IoMdClose
                            onClick={() => setSelectFormId(null)}
                            style={{ color: "red", fontSize: "25px" }}
                          />
                        ) : (
                          <FaChevronDown
                            onClick={() => setModalStatus(true)}
                            style={{ color: "red", fontSize: "20px" }}
                          />
                        )}
                      </li>
                    </>
                  )}
                </div>
                <hr
                  style={{
                    border: "1px solid rgba(0, 0, 0, 0.15)",
                    marginTop: "10px",
                  }}
                />
                <InputForm
                  style={{
                    opacity: formPramission?.queuing_function === 0 ? 0.5 : 1,
                  }}
                  disabled={
                    formPramission?.queuing_function === 0 ? true : false
                  }
                  label="Add Queing From"
                  type="radio"
                  value={"queing"}
                  name="form_type" // Adding the same name for grouping
                  register={{ ...register2("form_type") }}
                  onChange={handleFormTypeChange}
                />
                <InputForm
                  label="No From"
                  type="radio"
                  value={"noform"}
                  name="form_type" // Adding the same name for grouping
                  register={{ ...register2("form_type") }}
                  onChange={handleFormTypeChange}
                />
              </div>
            </div>

            <div className="Add_Company_Popup_form_colorpicker_div">
              <div
                className="Add_Company_Popup_form_colorpicker"
                htmlFor="favcolor"
              >
                {colorValue ? (
                  ""
                ) : (
                  <svg
                    width="30"
                    height="30"
                    viewBox="0 0 30 30"
                    fill="none"
                    xmlns="http://www.w3.org/2000/svg"
                  >
                    <path
                      d="M2.005 22.5027C2.65434 23.6231 3.45337 24.6664 4.39339 25.6066C7.22672 28.4397 10.9934 30 15 30C19.0068 30 22.7735 28.4397 25.6066 25.6066C26.5469 24.6664 27.3457 23.6231 27.995 22.5027C29.3026 20.2467 30 17.6772 30 15C30 12.3228 29.3026 9.75334 27.995 7.49725C27.3457 6.37688 26.5466 5.3334 25.6066 4.39339C22.7735 1.56029 19.0068 0 15 0C10.9934 0 7.22649 1.56029 4.39339 4.39339C3.45314 5.3334 2.65411 6.37688 2.005 7.49725C0.697403 9.75334 0 12.323 0 15C0 17.677 0.697403 20.2467 2.005 22.5027ZM11.172 15C11.172 14.3033 11.3599 13.6503 11.6865 13.0868C12.3493 11.9433 13.5857 11.1717 15 11.1717C16.4143 11.1717 17.6507 11.9433 18.3138 13.0868C18.6401 13.6501 18.8283 14.3033 18.8283 15C18.8283 15.6967 18.6401 16.3497 18.3138 16.913C17.6509 18.0567 16.4143 18.828 15 18.828C13.5857 18.828 12.3493 18.0567 11.6862 16.913C11.3599 16.3497 11.172 15.6967 11.172 15Z"
                      fill="#FF8398"
                    />
                    <path
                      d="M14.9961 24.4143V30.0004C19.0029 30.0004 22.7696 28.4401 25.6027 25.607C26.543 24.6667 27.3418 23.6235 27.9911 22.5031L23.15 19.708C21.5222 22.5214 18.4804 24.4143 14.9961 24.4143Z"
                      fill="#54E360"
                    />
                    <path
                      d="M24.4126 14.9998C24.4126 16.7148 23.9537 18.3227 23.1523 19.7075L27.9934 22.5026C29.301 20.2465 29.9984 17.677 29.9984 14.9998C29.9984 12.3226 29.301 9.75316 27.9934 7.49707L23.1523 10.2919C23.9537 11.6769 24.4126 13.2848 24.4126 14.9998Z"
                      fill="#008ADF"
                    />
                    <path
                      d="M6.84889 19.708L2.00781 22.5031C2.65715 23.6235 3.45618 24.6667 4.39619 25.607C7.22952 28.4401 10.9962 30.0004 15.0028 30.0004V24.4143C11.5185 24.4143 8.4767 22.5214 6.84889 19.708Z"
                      fill="#FFD400"
                    />
                    <path
                      d="M23.15 10.2921L27.9911 7.49725C27.3418 6.37688 26.5427 5.3334 25.6027 4.39339C22.7696 1.56029 19.0029 0 14.9961 0V5.58586C18.4804 5.58586 21.5222 7.47894 23.15 10.2921Z"
                      fill="#0065A3"
                    />
                    <path
                      d="M2.005 22.5026L6.84609 19.7075C6.04477 18.3227 5.58609 16.7148 5.58609 14.9998C5.58609 13.2848 6.04477 11.6769 6.84609 10.2922L2.005 7.49707C0.697403 9.75316 0 12.3228 0 14.9998C0 17.6768 0.697403 20.2465 2.005 22.5026Z"
                      fill="#FF9100"
                    />
                    <path
                      d="M15.0028 5.58586V0C10.9962 0 7.22929 1.56029 4.39619 4.39339C3.45595 5.33341 2.65692 6.37688 2.00781 7.49725L6.84889 10.2924C8.4767 7.47894 11.5185 5.58586 15.0028 5.58586Z"
                      fill="#FF4949"
                    />
                    <path
                      d="M18.827 14.9999C18.827 15.6966 18.6389 16.3496 18.3125 16.9129L23.1527 19.7075C23.954 18.3228 24.4129 16.7149 24.4129 14.9999C24.4129 13.2846 23.954 11.677 23.1527 10.292L18.3125 13.0866C18.6389 13.6499 18.827 14.3031 18.827 14.9999Z"
                      fill="#0065A3"
                    />
                    <path
                      d="M18.3098 13.0869L23.15 10.2924C21.5222 7.47902 18.4804 5.58594 14.9961 5.58594V11.1718C16.4104 11.1718 17.6468 11.9434 18.3098 13.0869Z"
                      fill="#005183"
                    />
                    <path
                      d="M14.9961 18.8289V24.4147C18.4804 24.4147 21.5222 22.5219 23.15 19.7085L18.3098 16.9141C17.647 18.0576 16.4104 18.8289 14.9961 18.8289Z"
                      fill="#00AB5E"
                    />
                    <path
                      d="M15.0016 24.415V18.8291C13.5873 18.8291 12.3509 18.0578 11.6878 16.9141L6.84766 19.7087C8.47546 22.5221 11.5173 24.415 15.0016 24.415Z"
                      fill="#FF9F04"
                    />
                    <path
                      d="M11.1718 14.9996C11.1718 14.3029 11.3597 13.6499 11.6863 13.0864L6.84593 10.292C6.04462 11.6767 5.58594 13.2846 5.58594 14.9996C5.58594 16.7146 6.04462 18.3225 6.84593 19.7073L11.6861 16.9126C11.3597 16.3494 11.1718 15.6964 11.1718 14.9996Z"
                      fill="#FF4B00"
                    />
                    <path
                      d="M15.0016 11.1718V5.58594C11.5173 5.58594 8.47546 7.47902 6.84766 10.2924L11.688 13.0869C12.3509 11.9434 13.5873 11.1718 15.0016 11.1718Z"
                      fill="#E80048"
                    />
                  </svg>
                )}

                <input
                  type="color"
                  id="favcolor"
                  name="favcolor"
                  value={colorValue}
                  style={{
                    visibility: colorValue ? "visible" : "hidden",
                    width: "30px",
                    height: "30px",

                    padding: "0px",
                    backgroundColor: "none",
                  }}
                  {...register("color")}
                  onChange={(e) => setColorValue(e.target.value)}
                />
                <label htmlFor="favcolor">Select your favorite color</label>
              </div>
              <div className="Add_Company_Popup_form_arrow_div">
                <label htmlFor="favcolor">
                  <svg
                    width="11"
                    height="20"
                    viewBox="0 0 11 20"
                    fill="none"
                    xmlns="http://www.w3.org/2000/svg"
                  >
                    <path
                      d="M1 1L10 10L1 19"
                      stroke="#FB6D48"
                      strokeWidth="2"
                      strokeLinecap="round"
                      strokeLinejoin="round"
                    />
                  </svg>
                </label>
              </div>
            </div>

            <div className="Add_Company_Popup_form_footer">
              <Button
                className="cancel"
                name={"Close"}
                type="button"
                fun={() => {
                  setSubGeofaceModel(false),
                    reset({
                      geofence_form_name: "",
                      form_type: "",
                      color: "",
                    }),
                    setSelectFormId(null),
                    setColorValue(null);
                }}
              />
              <Button name={"Save"} />
            </div>
          </form>
        </div>
      </Modal>

      <Modal status={modalStatus}>
        <div className="TableContain" style={{ padding: "0", height: "100%" }}>
          <table>
            <thead>
              <tr>
                <th>Sr.No</th>
                <th>Company Name</th>
                <th>Action</th>
              </tr>
            </thead>
            {data?.data?.data &&
              data?.data?.data.map((item, i) => (
                <tbody key={i}>
                  <tr>
                    <td>{i + 1}</td>
                    <td> {item.Company_name} </td>
                    <td className="table_icons">
                      <Button
                        name="Select"
                        fun={() => {
                          setSelectFormId({
                            id: item.id,
                            name: item.Company_name,
                          }),
                            setModalStatus(false);
                        }}
                      />
                    </td>
                  </tr>
                </tbody>
              ))}
          </table>
          <div
            className="Add_Company_Popup_form_footer"
            style={{
              display: "flex",
              justifyContent: "center",
              padding: "40px 0px",
            }}
          >
            <Button
              className="cancel"
              name={"Close"}
              type="button"
              fun={() => setModalStatus(false)}
            />
          </div>
        </div>
      </Modal>
    </>
  );
};

export default EditGeofence;
